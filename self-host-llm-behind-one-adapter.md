# Why I self-hosted the classifier model from day one, and how I hid the entire vendor behind a single Python file

The default instinct in 2026, when you need an LLM, is to call a hosted API. Zero infrastructure, best accuracy, one HTTP call. For the compliance classifier I'm building, the one that reads a free-text payment description and returns a regulatory code, I did the opposite. I self-hosted a small open model on my own machine from day one.

That decision could have been a trap. The obvious failure mode of self-hosting early is that you discover the small local model isn't accurate enough, and now you're rewriting the service to call a cloud API instead. So the real decision was never just "self-host." It was two decisions fused. Self-host now, and make switching to a cloud provider a one-file change. This piece is about the second half, because the second half is what made the first half safe.

## Why not just call the cloud API

Three constraints pushed against the default.

The first is regulated data. The payment description is customer transaction data inside a licensed remittance flow, and it gets retained for years. Shipping every description to a third-party inference endpoint in another country is a data-residency and vendor-risk question, not a free lunch. Keeping inference in-house removes a third-party-processor question from the compliance surface entirely.

The second is cost shape. This classifier fires on every single outward transfer before it can send. It sits on the critical path of the product's headline metric. Per-call API pricing on a high-frequency compliance gate is a recurring variable cost that scales with transfer volume. A self-hosted model is a fixed cost you've already paid.

The third is honest about what the project is for. The build doubles as the way I'm learning full-stack AI engineering, and self-hosting with a hand-written provider boundary is the skill I want, not an accident of being cheap.

None of those three says "never use a cloud API." They say "start self-hosted, and don't get married to it." Which is exactly the shape the architecture had to deliver.

## Reversibility is bought by a boundary, not by building both sides

Here's the principle the whole design rests on. I committed to self-hosting without committing to it permanently. That only works if switching is genuinely cheap, and "cheap" has to be structural, not a promise.

So everything that knows the vendor exists lives in exactly one file. The model server URL, the model name, the shape of the request, the HTTP call, the exact path into the response JSON, all of it is quarantined in a single adapter module. The web route doesn't know there's an Ollama. It knows there's an adapter.

The route is a single line of delegation:

```python
import ollama_adapter
from contract import ClassifyAnswer, ClassifyRequest

app = FastAPI(title="compliance-service")

@app.post("/classify", response_model=ClassifyAnswer)
async def classify(req: ClassifyRequest) -> ClassifyAnswer:
    return await ollama_adapter.classify(req)
```

That `import ollama_adapter` is the only place a vendor-ish name appears upstream of the adapter, and even here it's a filename, not the vendor's API. Re-point that import at a `claude_adapter` implementing the same signature and the route doesn't change. Nothing upstream changes.

The adapter itself is where all the vendor knowledge is concentrated, deliberately, so it's contained:

```python
OLLAMA_URL = "http://localhost:11434/api/chat"
MODEL = "qwen2.5:7b-instruct"
PROMPT_PATH = Path(__file__).parent / "prompt.json"
TIMEOUT = httpx.Timeout(120.0, connect=5.0)


async def _ask_ollama(messages: list) -> str:
    parcel = {
        "model": MODEL,
        "messages": messages,
        "format": ClassifyAnswer.model_json_schema(),
        "options": {"temperature": 0},
        "stream": False,
    }
    try:
        async with httpx.AsyncClient(timeout=TIMEOUT) as client:
            resp = await client.post(OLLAMA_URL, json=parcel)
    except httpx.RequestError as exc:
        raise OllamaUnreachable(f"could not reach the model server: {exc!r}") from exc

    if resp.status_code != 200:
        raise OllamaBadStatus(f"model server returned {resp.status_code}: {resp.text}")

    return resp.json()["message"]["content"]
```

Three things make this a boundary rather than just a function. The URL and model name are named constants at the top, the single swap point. The `async def classify(req) -> ClassifyAnswer` signature is the contract any future adapter re-implements. And the vendor-specific response path, the `resp.json()["message"]["content"]` that knows this particular server's reply shape, is contained here. A different provider's adapter parses that provider's shape and hands back the same string, so nothing downstream ever sees the difference.

The prompt is the third leg. It lives in a JSON file, loaded and edited as data, never compiled into code. Swapping providers doesn't recompile anything, and a provider with different prompting needs gets its own data file.

## Keeping the vendor out of the error labels too

The error handling is where this boundary either holds or quietly leaks, so it got the same treatment. The adapter raises named category labels that carry no HTTP knowledge and no vendor knowledge:

```python
class AdapterError(Exception):
    """Base for every error the adapter raises."""

class OllamaUnreachable(AdapterError):
    """Could not get any response from the model server: down, refused, or timed out."""

class OllamaBadStatus(AdapterError):
    """Model server replied with a non-200 status."""

class EmptyReply(AdapterError):
    """Model server returned an empty answer, even after one retry."""

class IncorrectData(AdapterError):
    """The answer would not fit the schema, even after one retry."""
```

The categories underneath the names are universal: unreachable, bad status, empty, schema miss. A cloud adapter would raise the same four for the same conditions. The web layer maps each label to an HTTP status, and that mapping is written against the labels, not against the vendor, so it doesn't move when the provider does.

I'll flag my own mistake here, because it's a real one. The labels are named `OllamaUnreachable` and `OllamaBadStatus`. The categories aren't Ollama-specific, but the names read like they are. The day a cloud adapter raises `OllamaUnreachable` is the day those names become a confusing smell. They should have been `ProviderUnreachable` from the start. It's cheap to rename now and awkward after a second adapter exists, which is exactly the kind of thing that's obvious in hindsight and invisible while you're typing.

## How you prove a boundary is real and not just aspirational

Anyone can claim their vendor is isolated. The question is whether you can demonstrate it. The demonstration here is the test suite. The entire service is testable with no model running and no network, because the only thing you have to fake is one object.

```python
def _install(monkeypatch, replies, raise_exc=None):
    """Swap in a fake model server. replies handed out one per call."""
    captured = {"bodies": []}
    state = {"i": 0}

    class _FakeClient:
        async def __aenter__(self): return self
        async def __aexit__(self, *e): return False
        async def post(self, url, json=None):
            captured["bodies"].append(json)
            captured["url"] = url
            if raise_exc is not None:
                raise raise_exc
            payload, status = replies[min(state["i"], len(replies) - 1)]
            state["i"] += 1
            return _FakeResponse(payload, status_code=status)

    monkeypatch.setattr(ollama_adapter.httpx, "AsyncClient", _FakeClient)
    return captured
```

That last line is the tell. The only thing the test fakes is the single `httpx` client inside the single adapter. If the vendor had leaked into the route, or the contract, or the error handling, you'd have to fake it in more than one place. You don't. One fake-point is the measure of how contained the vendor actually is, and one is what it takes.

There's a second test I like more than I expected to. It pins the exact request the adapter sends on the wire, so a future careless edit to the parcel shape fails loudly instead of silently changing what the model receives:

```python
async def test_parcel_has_the_five_locked_keys(monkeypatch):
    captured = _install(monkeypatch, [(SUCCESS_REPLY, 200)])
    await ollama_adapter.classify(_req())
    body = captured["bodies"][0]
    assert captured["url"] == ollama_adapter.OLLAMA_URL
    assert body["model"] == ollama_adapter.MODEL
    assert body["format"] == ClassifyAnswer.model_json_schema()
    assert body["options"] == {"temperature": 0}
    assert body["stream"] is False
```

The canned replies these tests replay aren't invented. They're the actual response shapes captured from real probes against the running model server, including the empty-reply-on-cold-load signature. So the tests run offline but they're grounded in how the real vendor actually behaves, which is the property that makes offline tests worth trusting.

## The model choice was about predictability, not benchmarks

A note on which model, because the reasoning runs against the grain of how models usually get picked. I chose `qwen2.5:7b-instruct` for what it lacks: a thinking mode.

The biggest structured-output gotcha in 2026 is reasoning models that emit thinking blocks before their answer. Those blocks break schema-constrained decoding, which is the exact mechanism keeping this classifier's output valid. This is corroborated across the inference-server ecosystem and the model vendor's own documentation, which states plainly that models in thinking mode don't support structured output. A newer, higher-benchmark reasoning model would score better on paper and break the JSON contract in practice. On a compliance gate, "never breaks the contract" beats "a few points higher on a benchmark." I picked the conservative, boring model on purpose.

## The runtime trap that cost real time

One war story, because it's invisible in the code and it's the kind of thing that eats an afternoon. Installing the model server the obvious way, through the standard package manager formula, gave me a broken setup. The formula shipped the API binary but not the actual inference runner. The server answered version checks and could not run a model. It looked installed. It wasn't, in the way that mattered.

The fix was to install the official application bundle instead, which ships the inference runner and the hardware acceleration together. What stuck with me wasn't the fix, it was the failure mode. The error message even suggested a build-from-source command, which carried its author's assumptions about who was hitting this. It was a maintainer's instruction, not an end-user's fix. Following the error's own suggestion would have sent me further down the wrong path. The packaging of the runtime turned out to be part of the architecture, not a footnote, and it lives nowhere in the source so it's exactly the kind of knowledge that evaporates unless you write it down.

## What I'd do differently

Write the second adapter's interface test before claiming reversibility. Right now the one-file-swap claim is proven by inspecting the seam, not by having done a swap. A short contract test asserting that any `classify(req) -> ClassifyAnswer` implementation satisfies the same input, output, and error-label contract would turn "reversible by design" into "reversible, verified," and it would catch the day a cloud provider's structured-output mechanism doesn't map cleanly onto this Ollama-shaped boundary. That last risk is real: the grammar-constraint guarantee that makes the output valid is provider-specific, so a cloud adapter would need that guarantee re-established on the new provider's mechanism, not assumed.

Adopt the official client library now, keep the adapter. Raw `httpx` was the right teaching choice, but it re-implements retry, timeout, and error-type semantics the library gives for free, including a total-duration timeout across the whole retry chain that the current per-call timeout doesn't bound. The adapter boundary is precisely what makes this swap cheap. I'd take the library and keep the seam.

Promote the URL and model name to environment variables the moment a second environment exists. Hardcoded constants are fine for one machine, but the reversibility story has a hole until "which provider, which endpoint" is configuration rather than source. It's a ten-minute change, deferred for a good reason today, and it should be the first thing done when deployment starts rather than discovered later.

Write the model-swap trigger down as a threshold, not a vibe. "Switch to a bigger or cloud model if accuracy is insufficient" needs a number to fire on. The boundary that enables the swap is ready. The decision rule that should trigger it is undefined, and it stays undefined until there's an evaluation set to measure against, which is still the biggest gap across this whole project.

## The shape of it

One file knows the vendor. The route delegates to it blindly. The error labels carry neither HTTP nor vendor knowledge, so they survive a provider swap unchanged. The prompt is data. The whole service tests offline because there's exactly one object to fake. And the model was chosen for the boring property of never breaking its output contract rather than for its benchmark score.

The honest limit is that the reversibility is a design proven by inspection, not a fallback I've exercised. The second adapter doesn't exist. But the value was never in building both sides up front. It was in committing to self-hosting without that commitment being a one-way door, and a boundary you can demonstrate with a single fake-point in the tests is what makes the door swing both ways.
