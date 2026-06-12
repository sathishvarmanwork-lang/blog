# Making an invalid answer physically impossible: grammar-constrained LLM output for regulated classification

I built a classifier that maps a free-text payment description to a single Bank Negara Malaysia (BNM) foreign-exchange purpose code. The hard requirement was not accuracy. It was something stricter: the model must be incapable of emitting a code that doesn't exist in the official taxonomy.

This is the difference between an LLM that's usually right and an LLM that's safe to put behind a regulated declaration. This is a long writeup, because the interesting parts are in the details: a hallucinated number that decided the whole design, a confidence field I deleted, and an evaluation harness that quietly lied to me for a while.

## The problem

Malaysia's Foreign Exchange Administration rules require a purpose-of-payment code on every outward transfer. A user types a free-text description, something like "200 pairs of leather shoes from a maker in Wenzhou," and the system has to map that to exactly one code from a fixed BNM taxonomy.

The mapping is genuinely hard. Apparel and footwear belong to one code; the "manufactured goods classified by material" code looks superficially similar but is wrong for finished articles. A human compliance officer gets this right with training. A naive keyword lookup does not.

An LLM is the right tool for the mapping. But a raw LLM is dangerous for the output, because a regulated declaration cannot contain a code that doesn't exist. The very first thing I tested proved that danger was real.

## The first probe, and the number that decided everything

Before writing any structured-output code, I ran the simplest possible version: plain prompt, ask the model for the code in prose, parse it out.

The model returned a fluent, confident answer. Here is the actual output, lightly trimmed:

```
…Purpose Code 4219 - "Imports of Goods" is typically used to categorize
payments related to the purchase and importation of tangible goods…
- BNM Purpose Code: 4219
```

The code `4219` does not exist anywhere in the BNM taxonomy. The model invented a plausible-looking number and explained it with total confidence.

No parser can fix an invented value. You cannot regex your way out of a hallucinated declaration. That single probe killed the entire "prompt and parse" category and set the real requirement: invalid codes have to be impossible at the point of generation, not filtered out afterward.

## Why the obvious fixes don't work

Once you accept "impossible, not filtered," you can walk through the tempting alternatives and watch each one fail.

Loose JSON mode. Most local model servers offer a "return valid JSON" mode. This guarantees the output is syntactically valid JSON. It does not guarantee your fields, and it does not guarantee your value set. `{"purpose_code": "4219"}` is perfectly valid JSON. This solves nothing that matters here.

A schema with a free-string field. Better. Now you get structured output with known fields. But if the code field is typed as a string, the value is still unconstrained. A hallucinated string passes the schema. Half a solution.

Listing the codes in the prompt only. You can enumerate every legal code in the system prompt and instruct the model to use only those. But prompt instructions are advisory. The model can and will ignore them under the right input. Constraint has to be applied to the model's actual output distribution, not requested politely in natural language.

Fine-tuning a dedicated classifier. Real option, wrong stage. No labeled dataset existed, and training a model is dramatically heavier than the problem needs when a constraint mechanism already exists. Parked, not killed.

## The decision

Make the purpose code a closed enum. Generate a JSON Schema from a Pydantic model. Pass that schema to the model server's structured-output parameter so that generation is grammar-constrained: at every token step, the sampler masks any token that would break the schema. Then re-validate the result with Pydantic as a second gate.

The contract is two fields and nothing else:

```python
from enum import StrEnum
from pydantic import BaseModel, ConfigDict

class PurposeCode(StrEnum):
    FOOD = "00000"
    BEVERAGES_TOBACCO = "01000"
    CRUDE_MATERIALS = "02000"
    MINERAL_FUELS = "03000"
    OILS_FATS = "04000"
    CHEMICALS = "05000"
    MANUFACTURED_GOODS = "06000"
    MACHINERY_TRANSPORT = "07000"
    POWER_LINES = "07100"
    MISC_MANUFACTURED = "08000"
    MISC_NEC = "09000"
    NON_MONETARY_GOLD = "09700"
    NONE_FITS = "NONE_FITS"

class ClassifyRequest(BaseModel):
    description: str

class ClassifyAnswer(BaseModel):
    model_config = ConfigDict(extra="forbid")
    reasoning: str
    purpose_code: PurposeCode
```

A few details in here are load-bearing.

`StrEnum` makes each member's value the literal code string (`"00000"`) and its name the human-readable label (`FOOD`). Because the values are strings, leading zeros survive. An integer enum would silently turn `00000` into `0`.

`ConfigDict(extra="forbid")` matters more than it looks. The auto-generated schema initially did not include `additionalProperties: false`, which would have allowed the model to smuggle in extra keys. I caught this by diffing the generated schema against what I intended. The config closes the gap.

`reasoning` comes before `purpose_code` in the field order, and has no length cap. Order matters because the model generates top-to-bottom; letting it reason first tends to produce a better-grounded final code. A length cap is a trap, because truncating mid-string produces invalid JSON the grammar can't recover from.

When you call `model_json_schema()` on that answer model, it produces exactly the object the model server uses to build its grammar:

```json
"format": {
  "type": "object",
  "properties": {
    "reasoning": { "type": "string" },
    "purpose_code": {
      "type": "string",
      "enum": ["00000","01000","02000","03000","04000","05000","06000",
               "07000","07100","08000","09000","09700","NONE_FITS"]
    }
  },
  "required": ["reasoning", "purpose_code"],
  "additionalProperties": false
}
```

The `enum` array is where the constraint physically lives. The server masks any token that can't continue one of those thirteen strings, so `"4219"` is unreachable. `additionalProperties: false` is the wire form of `extra="forbid"`, and `required` forces both fields present.

The full request envelope to the model server is five keys, and the temperature is zero because this is classification, not generation:

```jsonc
{
  "model": "qwen2.5:7b-instruct",
  "messages": [ /* system + 4 few-shot pairs + live user turn */ ],
  "format":  { /* the schema above */ },
  "options": { "temperature": 0 },
  "stream":  false
}
```

The enum is the lock. With grammar-constrained decoding, the `4219` from probe zero is now physically unreachable. No token path spells a non-member string.

## Grammar constrains structure, not values

Here is the single most important thing I learned, and it cuts both ways.

The win. The set of legal codes is finite, thirteen members, so the grammar can mask every token that doesn't spell one of them. The "is this a legal code?" question is now airtight. `4219` cannot happen.

The residual risk. The grammar cannot guarantee the right code among the legal ones. A confident, structurally-perfect `06000` for a t-shirt that should be `08000` is exactly as schema-valid as the correct answer. The enum bought validity. In doing so, it exposed correctness as a completely separate problem.

This is the honest center of the whole project. The enum was necessary and nowhere near sufficient. Solving validity made it obvious that accuracy needed its own, different solution.

## Teaching the boundary

So how do you get the correct legal code? My first instinct was rules: write explicit instructions in the prompt to disambiguate the tricky boundaries. This turned into whack-a-mole, and the probe ladder shows exactly how:

| Probe | Setup | Result | Lesson |
|---|---|---|---|
| P0 | No schema, plain prompt | prose + hallucinated `4219` | A regulated code field can't be free text. |
| P1 | Official definitions, no examples | t-shirt → `06000` (wrong) | Enum gives a legal code, not a correct one. |
| P2 | + a redirect hint | reasoning flipped right, code still `06000` | Reasoning and final answer can disconnect. |
| P3 | + a hard rule "apparel always 08000" | t-shirt → `08000` (right) | One rule fixed one case. |
| P4 | broad rule "material vs finished" | chairs right, but steel → wrong | And broke a neighbor. Abstract rules are whack-a-mole. |
| P5 | few-shot examples instead of rules | 4/4 held-out correct | Examples generalize without side effects. |

The turn at P4 to P5 is the lesson. Every abstract rule I added to fix one category quietly broke an adjacent one. The "material vs finished article" rule that correctly classified office chairs as finished goods also pushed steel into the wrong bucket. Concrete few-shot examples generalized cleanly without the side effects.

The prompt ended up carrying four few-shot pairs, each teaching one trap as a real conversation turn. The assistant side of each turn is a JSON string in the exact answer shape, which also shows the model what well-formed reasoning looks like.

The apparel pair, which fixes the systemic confusion between the material code and the finished-article code:

```json
{ "role": "user",
  "content": "Payment description: 200 pairs of leather shoes from a maker in Wenzhou." },
{ "role": "assistant",
  "content": "{ \"reasoning\": \"Leather shoes are footwear, a finished-article type under 08000. Not 06000, which classifies goods by material rather than finished article.\", \"purpose_code\": \"08000\" }" }
```

The other three pairs each guard a different boundary:

- Cold-rolled aluminium sheet maps to the manufactured-goods-by-material code. It is worked material, not a finished article, and not raw ore. This guards the opposite side of the apparel boundary.
- Unprocessed iron ore maps to the crude-materials code. This is the over-correction guard, so that "raw versus finished" reasoning doesn't drag genuinely worked material like the aluminium back down into raw materials.
- Audit and accounting fees map to `NONE_FITS`. This is not goods at all, so the right answer is "send to a human," not a goods code.

After switching to few-shot, a batch of four held-out cases never shown to the model all classified correctly: office chairs to the finished-goods code, steel coils to the worked-material code, consulting fees to `NONE_FITS`, coffee beans to the food code. Four traps, four passes, no rule-based side effects.

## The adapter, mid-build

I want to show the adapter honestly, including the fact that it isn't finished. The classifier is validated against the model server through direct probes; the HTTP service wiring is still in progress. Here is the current on-disk shape of the one file that knows which model vendor is behind the call:

```python
import httpx, json
from contract import ClassifyAnswer, ClassifyRequest

OLLAMA_URL = "http://localhost:11434/api/chat"
MODEL = "qwen2.5:7b-instruct"

async def classify(req: ClassifyRequest) -> ClassifyAnswer:
    messages = json.load(open("prompt.json"))
    # NEXT: append the live turn, build the request
    #   {model, messages, format=ClassifyAnswer.model_json_schema(),
    #    options{temperature:0}, stream:False}
    # THEN: httpx POST, then ClassifyAnswer.model_validate_json(...) with retry
```

Two deliberate choices are already visible. The model URL and name are named constants at the top, because the whole point of isolating this in one file is that swapping to a cloud API later touches exactly one place. And `classify` is `async` on purpose: a production service should serve other requests while one classification waits on the model.

The reason this file exists as a boundary at all is reversibility. The model vendor is hidden behind this single adapter; the prompt is swappable data, not code. Moving from a self-hosted model to a hosted API later is a one-file change, not a rewrite.

## When the eval harness almost lied to me

This is the part I almost didn't write down, which is exactly why it belongs here.

The evaluation harness loads a locked request file (system prompt, plus the four few-shot example pairs, plus one final user turn) and swaps only the last turn for each test case. The correct way to do that is to keep everything except the last message and append the new one:

```python
req["messages"] = base["messages"][:-1] + [{"role": "user", "content": payload}]
```

An earlier version of the harness rebuilt the message list as `[system, payload]` instead. It kept the system prompt and the test case, and silently dropped all four few-shot examples.

The harness was reporting PASS. But it was testing the prompt without the few-shot examples, which is a different, weaker prompt than the one actually shipping. Every green result was measuring the wrong configuration. The thing I was using to validate the prompt had a bug that invalidated its own results, and it did so quietly, with no error and a reassuring row of passes.

The lesson generalizes well past this project. Your evaluation infrastructure is code, and it can be wrong in exactly the way that hides from you: a passing test against the wrong setup looks identical to a passing test against the right one. The test rig needs its own scrutiny. After this, I stopped trusting any "PASS" until I'd confirmed the harness was assembling the real prompt.

One more small choice in the harness: it uses only the Python standard library for HTTP, no third-party client, so it runs anywhere without the service's dependencies. An eval rig with zero dependencies is one less thing that can drift away from working.

## The field I deleted

The original contract had a third field: `confidence`, a 0-to-1 number the model would self-report. The idea was to route low-confidence answers to human review.

Two problems killed it. The grammar can enforce that the field exists and is a number, but it cannot enforce a numeric range; range-checking would have to be application code regardless. More decisively, here is an actual early response while the field was still in the schema:

```json
{"message":{"content":"{ \"reasoning\": \"…classified as manufactured goods
  (06000) … specifically as part of the apparel, clothing, bags and footwear
  category.\", \"purpose_code\": \"08000\", \"confidence\": 1 }"}}
```

Look at what that reasoning does. It names the wrong code (`06000`) and the right code (`08000`) in the same breath, contradicts itself, and reports confidence `1`. The same `confidence: 1` showed up on a flatly wrong answer in a sibling probe. Identical signal for opposite correctness.

A self-reported confidence that's maxed out whether the answer is right or wrong is not a confidence signal. It's noise wearing a number. So I deleted it. Self-reported confidence from an LLM is not calibrated confidence, and specifying the field before deciding what kind of confidence you actually need is a round-trip you can skip.

What I kept instead was a meaningful distinction at the enum level. A "miscellaneous goods" code for "this is goods but fits no specific category," kept separate from a dedicated `NONE_FITS` member for "I can't tell what this is, send it to a human." Two different signals compliance genuinely needs, encoded structurally rather than via an untrustworthy number.

## The second gate

The model server applies the grammar during generation, but it does not re-validate the complete response against the schema afterward. A truncated stream can yield unclosed JSON that the grammar alone won't catch.

So the client re-validates every response with `model_validate_json`. On a validation error, retry while feeding the error back to the model. After a few attempts, fail loud with a clear error rather than clamping to a default. A compliance gate should never silently substitute a guess. The enum is necessary; the second validation gate is what makes it sufficient.

## The error contract

Because this sits in front of a regulated transfer, the failure modes need to be as defined as the success path. I ran a set of probes feeding the model server malformed and degenerate inputs, then mapped each observed failure to the HTTP status the service should return:

| Condition | Server response | Service returns |
|---|---|---|
| model name typo | not-found error | 502 |
| malformed request body | parse error | 502 |
| empty message list | error, no usable content | 502 |
| cold model, empty generation | 200, empty content, "load" reason | validate, retry, then 422 |
| pure garbage description | 200, valid `09000` | use it |

The last row is the important positive result. Fed a description of pure junk with a stripped-down prompt, the grammar still produced a legal enum code. It cannot emit a non-code. (It returned the miscellaneous-goods code rather than `NONE_FITS` only because that probe used a minimal prompt without the few-shot guidance, which is an accuracy nuance, not an error-contract bug.)

The whole contract compresses to one line: server down maps to 502; a non-200 error maps to 502; a 200 with empty content gets validated, retried, then fails as 422; a 200 with valid content is used.

## Service setup

The service is dependency-light on purpose, managed with a standard `pyproject.toml`. Pydantic rides inside FastAPI rather than being pinned separately:

```toml
[project]
name = "classification-service"
requires-python = ">=3.14"
dependencies = [
    "fastapi>=0.136.3",
    "httpx>=0.28.1",
    "uvicorn>=0.49.0",
]
```

The model itself is Qwen2.5-7B-Instruct, Q4_K_M quantization, 4.7 GB on disk and roughly 5 to 7 GB resident, running on Metal on an Apple M2 with 11.8 GiB of unified memory. I chose this specific model partly because it has no thinking mode. Models that emit reasoning blocks before their answer are a known trap for schema-constrained decoding, because those blocks break the grammar. A model that goes straight to structured output sidesteps the problem entirely.

## What I'd do differently

Separate "legal" from "correct" on day one. The biggest time sink was discovering, probe by probe, that the enum bought validity but not accuracy. Stating that distinction upfront would have sent me straight to few-shot plus a held-out set, instead of burning probes P1 through P4 chasing rule tweaks.

Assert the generated schema instead of eyeballing it. `model_json_schema()` silently omitted `additionalProperties: false` until I diffed it by hand. A one-line test that asserts the generated schema matches the intended contract would catch schema drift on a compliance field automatically, which is exactly the kind of thing that should fail in CI rather than in someone's eyes.

Don't spec a field before deciding what it means. The confidence field was designed in, then deleted once I realized self-reported isn't calibrated. The honest signal, token-level log-probabilities, is a different mechanism entirely, and if I want real confidence later that's where it comes from, not from asking the model how sure it feels.

Write the eval set before the prompt, not after. All the accuracy evidence here is hand-probes. A proper held-out set of thirty to fifty cases covering the known traps (apparel, furniture, raw versus worked metal, services, gold) would let me measure prompt changes instead of eyeballing four cases at a time. The harness bug is a second argument for this: a fixed, asserted prompt prefix would have caught the dropped few-shot examples on the first run.

## What's actually next

The final design is small and boring in the best way: a two-field contract, a thirteen-member enum, grammar-constrained decoding, four few-shot examples, a client-side validation gate with explicit retry and loud failure, and a defined error contract. No fine-tuning. No heavy framework.

But the eval set is the obvious gap, and a real one would probably surface edge cases the small probe ladder hasn't seen. The system can't lie about the codes themselves anymore. Whether it picks the right ones at scale is the next thing to measure, and the next thing worth being wrong about in public.

---

*A note on the numbers: the probe results above are single-digit hand-tested cases from the build loop, not a statistical evaluation. They're directional evidence for the design decisions, not a benchmark.*
