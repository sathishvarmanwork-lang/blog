# Two honest states: the error contract for an LLM behind a compliance gate

In an earlier piece I made an invalid output impossible: a closed enum plus grammar-constrained decoding so the model physically cannot emit a payment code that doesn't exist. That solved one thing. It also created a quieter problem I'd half-ignored.

"The model can't emit `4219`" is not the same as "the call always succeeds." The grammar governs what tokens the model may produce. It says nothing about what happens when the model server is down, when it times out mid-generation, when it returns a 200 with an empty body because it was still loading into memory, or when a truncated stream hands back JSON the grammar never got to close. The classifier sits in front of a regulated cross-border transfer. Every one of those failure modes needs an answer, and the wrong answer to any of them is the same wrong answer: paper over it and ship something.

This piece is about the contract for going wrong. It has one principle and a lot of consequences.

## The principle: there is no third state

A compliance answer has exactly two honest states. Either it is validated and correct, or it is an explicit "we could not classify this, send it to a human." There is no third state where the system invents a plausible code to keep the request moving.

That sounds obvious written down. It is not how software usually behaves under failure. The default instincts of a careful engineer all point the wrong way here. Catch the error and return a sensible default. Retry until it works. Clamp the value to the nearest legal one. Every one of those is a way of manufacturing the third state, and every one of them, behind a regulated declaration, is worse than failing.

A purpose code the system fabricated to cover a failed model call is a false regulatory declaration with a seven-year retention tail. A loud failure routes to a compliance officer. A silent default ships the lie. So the entire error design is built backwards from a refusal: never coerce, never default, never retry blindly. Once you commit to that, the rest is just working out what each failure should become instead.

## What I rejected, and why

Before the design that shipped, here are the tempting alternatives, each of which manufactures the third state or leaks something it shouldn't.

Clamp or coerce. On a bad value, snap to the nearest legal code or a default. This is the third state in its purest form. Disqualifying on its own terms.

Blind retry. On any failure, just call again. This one is worse than it looks, because the model runs at temperature zero. At temperature zero the model is deterministic, so re-running the exact same messages reproduces the exact same bad output. A blind retry isn't a fix, it's the same failure again at the cost of another round trip, plus a retry storm against a single local model.

Let raw exceptions bubble. No handlers, let the HTTP and JSON errors turn into a 500 with a stack trace. On a payments service this leaks the model server's address, the model name, and internal file paths to the caller. That's an information-disclosure defect, not an error strategy.

HTTP status codes raised inside the adapter. Have the code that talks to the model raise an HTTP 502 directly. Tidy, and wrong, because it welds the adapter to the web framework. The moment you want to reuse that classifier in a CLI, a batch job, or an agent node, it drags the entire HTTP stack along with it.

One generic error type. Raise a single "adapter failed" and return one status for everything. This throws away the only distinction that matters under failure: whose fault was it, and what should the caller do next.

## The distinction that matters: whose fault is it

That last point is the whole game, so it gets its own section. When the call fails, the caller needs to know which of three different worlds they're in, because the right action differs in each.

The upstream genuinely failed. The model server is down, refused the connection, or timed out. Nothing was wrong with the request; the dependency is unavailable. The honest status is 502, and the honest message is "temporarily unavailable, try again."

We sent a bad request. The server returned a non-200 because our call was malformed: a typo'd model name returning a 404, a trailing comma returning a 400, an empty messages array. This is our bug, not theirs and not the merchant's. The honest status is 500, and the body says nothing more than "internal error," with no echo of the underlying 404 or 400.

We asked a fair question and couldn't get a usable answer. The request was fine, the server was up, but the model returned something empty or unparseable even after a retry. Nobody did anything wrong; the model just couldn't classify this input. The honest status is 422, and the answer routes to manual review.

Three failures, three worlds, three actions. Collapse them into one status and you've told the caller nothing they can act on.

## The bug that taught me the distinction

Here's the part worth being honest about, because the distinction above didn't arrive fully formed. During the probe phase, when I was testing failure modes with hand-run curl commands, I mapped every non-200 from the server to a 502. Server gave back anything that wasn't a 200, return a gateway error. Clean, uniform, and quietly wrong.

It only got sharp when I sat down to write the actual handler. A 502 means "the upstream failed." But a 404 for a model-name typo isn't the upstream failing. The upstream worked perfectly and correctly told me my request was nonsense. That's a 500, my own bug, not a gateway error. The non-200 mapping drifted from 502 to 500 in the gap between probing and building, and the drift was the design getting more correct, not less.

The lesson generalizes past this one status code. The rule, stated cleanly, is: 502 means the upstream failed, 500 means we sent a bad request, 422 means we asked a fair question and couldn't get a usable answer. Writing that sentence down before mapping the first failure would have skipped the drift entirely. I had the failure modes catalogued from the probes. What I didn't have, until the code forced it, was the one-line theory of fault that assigns each one a status.

## Retry once, and feed the mistake back

Two failure modes are genuinely transient and deserve exactly one retry. Not zero, not three, not exponential backoff. One.

The first is the empty-on-load case. A model freshly loaded into memory sometimes returns a 200 with empty content and a "load" reason: it finished loading instead of generating. The fix is a single immediate retry against the now-warm model, which usually succeeds. If it comes back empty twice, that's an `EmptyReply`, and it routes to review.

The second is the invalid-output case, and it's the one where the temperature-zero detail bites. If the model returns JSON that doesn't validate, you cannot just call again, because at temperature zero the identical messages produce the identical bad output. The retry has to change the conversation. So the retry appends the model's own bad answer as an assistant turn, then a user turn saying the last reply was invalid and to return only valid JSON matching the schema. That changed context is what makes the second attempt meaningfully different from the first rather than a deterministic replay of the same mistake. If it still doesn't validate, that's `IncorrectData`, and it also routes to review.

In code, both retries live inside the `classify` function. The empty-on-load retry is the immediate re-call; the invalid-output retry is the one that appends the bad answer and the correction:

```python
answer = await _ask_ollama(messages)

# Empty reply = the model had just finished loading; one immediate retry on the warm model.
if not answer.strip():
    answer = await _ask_ollama(messages)
    if not answer.strip():
        raise EmptyReply("Ollama returned an empty reply twice")

try:
    return ClassifyAnswer.model_validate_json(answer)
except ValidationError:
    # Temperature 0: a blind retry repeats the same bad output, so feed the mistake back.
    retry_messages = messages + [
        {"role": "assistant", "content": answer},
        {
            "role": "user",
            "content": "Your last reply was invalid. Reply with ONLY valid JSON matching the schema: {reasoning, purpose_code}.",
        },
    ]
    answer = await _ask_ollama(retry_messages)
    try:
        return ClassifyAnswer.model_validate_json(answer)
    except ValidationError as exc:
        raise IncorrectData(f"Ollama's reply did not match the schema: {exc}") from exc
```

The correction message is short and prescriptive on purpose, naming the schema fields explicitly so the model has a concrete target to hit on the second try.

One retry per class is a reasonable default, and I want to be honest that it's a default, not a measurement. I don't have data on how often the second attempt actually recovers versus needs a third, because I haven't logged attempts-versus-successes per class yet. "Retry once" is a guess dressed up as a decision. A small counter would turn it into a tuned number. That's on the list.

## Keeping the transport out of the labels

One structural choice holds the whole thing together. The code that talks to the model raises plain, HTTP-free error labels: unreachable, bad-status, empty, incorrect. Four named exceptions that know nothing about the web. A separate layer, the FastAPI app, owns the entire mapping from label to HTTP status, in app-wide handlers, so the route body stays a single line that calls the adapter and lets the label bubble up.

The label module itself is short and is the contract:

```python
"""Adapter-raised error labels.

The adapter raises these; the FastAPI endpoint catches them and maps each type
to an HTTP status code. They carry NO HTTP knowledge, which keeps the adapter
reusable off the web path.

    OllamaUnreachable  -> 502  (server down, refused, or timed out)
    OllamaBadStatus    -> 500  (Ollama returned a non-200; our request was wrong)
    EmptyReply         -> 422  (empty answer, even after one retry)
    IncorrectData      -> 422  (answer did not fit the schema, even after one retry)
"""


class AdapterError(Exception):
    """Base for every error the adapter raises."""


class OllamaUnreachable(AdapterError):
    """Could not get any response from Ollama: down, refused, or timed out."""


class OllamaBadStatus(AdapterError):
    """Ollama replied with a non-200 status (e.g. 400 bad request, 404 model not found)."""


class EmptyReply(AdapterError):
    """Ollama returned an empty answer, even after one retry."""


class IncorrectData(AdapterError):
    """Ollama's answer would not fit the schema, even after one retry."""
```

The shared `AdapterError` base lets a caller catch "any adapter failure" in one `except` when that's what they need, while still allowing per-type handlers when it isn't. The docstring doubles as the human-readable map between labels and statuses, so the contract sits next to the types it governs.

This split is what keeps the classifier reusable. The adapter decides what failed. The web layer decides what the outside world sees. Because the adapter carries no HTTP knowledge, the same classifier drops into a CLI or a batch job or an agent node without dragging a web framework behind it. And because the mapping lives in one place, the policy (which failure becomes which status, with which safe message) is one file you can read top to bottom, not a behavior scattered across every error site.

The mapping is four exception handlers, all going through one helper that owns the response shape:

```python
def _safe(status_code: int, message: str) -> JSONResponse:
    return JSONResponse(status_code=status_code, content={"detail": message})


@app.exception_handler(OllamaUnreachable)
async def _on_unreachable(request: Request, exc: OllamaUnreachable) -> JSONResponse:
    return _safe(502, "Classification service temporarily unavailable, please try again.")


@app.exception_handler(OllamaBadStatus)
async def _on_bad_status(request: Request, exc: OllamaBadStatus) -> JSONResponse:
    return _safe(500, "Internal error.")


@app.exception_handler(EmptyReply)
async def _on_empty(request: Request, exc: EmptyReply) -> JSONResponse:
    return _safe(422, "Could not classify this description, needs manual review.")


@app.exception_handler(IncorrectData)
async def _on_incorrect(request: Request, exc: IncorrectData) -> JSONResponse:
    return _safe(422, "Could not classify this description, needs manual review.")
```

`_safe()` is the single choke point that guarantees the body shape, so no handler can accidentally serialise an exception object. `EmptyReply` and `IncorrectData` map to the same status and the same message, kept as two labels so server logs can still tell empty-output and unparseable-output apart.

The safe messages cost something worth naming. The handler bodies are deliberately uninformative: "temporarily unavailable," "needs manual review." The real cause lives in the typed exception and goes to server logs, never to the HTTP body. The tradeoff is that the caller can't self-diagnose from the response. On a payments service, a caller that can't read your internals from an error body is the point, so I took that trade without much hesitation.

## What the tests can and can't prove

The error logic is covered by sixteen offline unit tests, all passing. They swap the HTTP client for a fake that records each outgoing request and hands back canned replies whose shapes were copied from real probes. Every branch, every mapping, every "no internals in the body" assertion is pinned. Two of those assertions are security checks I wanted executable rather than assumed: that the server address never appears in the 502 body, and that the underlying 404 never appears in the 500 body.

But I want to be precise about what that coverage means, because it's easy to oversell. The tests prove the logic, deterministically and forever. They do not prove that the real model server actually returns the shapes the fakes imitate. That evidence comes from single hand-captured curl probes: directional for the model's behavior, authoritative only for the shape of each failure. And there is still no automated integration test that boots the app, points it at a genuinely dead port, and watches the whole chain from connection error to 502 run end to end through the real HTTP layer. The pieces of that chain are each asserted; the chain as a whole is not. That's the most honest gap in the current state, and closing it is a single integration test I haven't written yet.

## What I'd do differently

Write the theory of fault before the first mapping. The 502-to-500 drift was the rule getting sharper, but it was also avoidable. One sentence assigning each failure class a status, written before any handler, would have caught it.

Make "needs manual review" a real product surface, not a 422 string. Both empty and invalid resolve to the same 422 and the same message, which is correct as an error contract. But what the compliance officer actually sees, how the merchant is told, and whether the failed description is queued anywhere are all undesigned. The error contract is solid. The human workflow that consumes a 422 is a stub. The unhappy paths all currently dead-end at a handoff nobody has built yet.

Assert the error map in a test, not in three docstrings. The label-to-status contract currently lives in a docstring, in the handlers, and in my notes, kept in sync by hand. A tiny table-driven test iterating the handlers would make the map executable, so a future edit that breaks it fails CI instead of silently disagreeing with the documentation.

## The shape of it

Strip away the specifics and the design is small. Four typed failure labels with no web knowledge. One retry per transient class, with the invalid-output retry feeding the mistake back to break determinism. A label-to-status mapping in one place, with bodies that leak nothing. And underneath all of it, a refusal: a compliance answer is validated or it is "send this to a human," and there is no plausible-looking third option the system is allowed to invent.

The grammar made the model unable to lie about the codes. The error contract makes the service unable to lie about its own failures. Those are two different kinds of honesty, and a regulated system needs both.
