# Why I stopped guessing the core problem my product solves, and how I now pick the wedge from evidence instead

Most product decisions I've watched go wrong didn't go wrong at the build stage. They went wrong at the first sentence, the one that names the single problem the product exists to solve. That sentence usually gets written in a kickoff room, from intuition, by whoever is most senior or most confident in the moment. Then the metric, the roadmap, and a year of engineering all line up behind it. If it's wrong, none of that work can be right, and nobody finds out for months.

So on a recent greenfield product I did something that felt like stalling at the time. I refused to write that sentence at kickoff. I parked the core-problem decision, ran the competitor research first, and only locked the problem afterward, against evidence. People in the room wanted the crisp problem statement on the wall, and not giving it to them felt like dodging the one thing I was there to do. I did it anyway, and I'd do it again. This is the method, the rule underneath it, and the part most product writeups skip, which is how thin my own evidence actually was.

## Parking the decision is itself the decision

The instinct in a kickoff is to define the problem immediately. It feels like progress. A crisp problem statement on the wall lets everyone start moving, and movement feels like the job.

The trouble is that an early problem statement is sticky in a way that's brutal to undo. Once it's written, the metric gets designed around it, the first features get scoped against it, and the team's mental model sets like concrete. By the time evidence shows up saying the problem was mis-stated, unwinding it means unwinding everything built on top. The cost of being wrong doesn't stay still. It compounds with every decision that leans on it.

So my first real decision was to not decide yet, and to be honest that this is a decision and not an absence of one. I split two things that usually get fused. Who the customer is, and what their core pain is. The customer I was willing to name early, because it's a smaller bet and cheap to correct if I got it wrong. The core pain I deliberately left open, with one rule nailed to it. It has to come from evidence, not from the room. That rule is the whole discipline. Without it, "we'll figure out the pain later" just means "we'll guess it later, with less energy than we have now."

## The rule I choose by is hurts most and least solved

When the evidence came in, there wasn't one obvious candidate. There were four, and they formed a ladder where any rung could plausibly have been the answer. The job was to choose, and the rule I chose by is the part worth stealing from this whole writeup.

The wrong rule is the one that feels right. Pick the pain that's most strategically important, or the one the team is most excited to build for, or the one that's easiest to measure. Each of those is a trap. Strategic importance is usually just the founder's prior wearing a suit. Excitement tracks what's fun to build, not what's worth building. And easy-to-measure quietly drags you toward the shallow pains, because the shallow ones are the legible ones.

The rule I actually used has two conditions welded together with an AND. The pain has to hurt the most, and it has to be the least solved by the people already in the market. Drop either half and it breaks. Hurts-most without least-solved drops you into a crowded lane where the incumbents already run hard, so your win is expensive to earn and trivial for them to copy. Least-solved without hurts-most lands you in an empty lane nobody cares about, which is empty for a reason. The only place a new entrant has solid ground is the intersection. A pain that genuinely hurts and that genuinely nobody has addressed.

Three of my four candidates failed the AND. They were real pains, but they were the pains the incumbents already competed on head-on. Solving them meant beating established players on their best ground, on dimensions they could match next quarter without breaking a sweat. The fourth was the loudest complaint in the research and the one nobody had addressed well. It won, and I want to be precise about why it won. Not because it was the most exciting. Because it was the only candidate standing in the intersection.

## The crowded lane is the comfortable mistake

Underneath the selection rule sits a tradeoff that deserves its own name, because it's what the rule is really optimising for.

A crowded lane is a pain every competitor already attacks. Everyone publishes their rates, so price transparency is crowded. Everyone is already fast, so speed is crowded. Standing out in a crowded lane is expensive, and any lead you get is fragile, because the incumbents are right there and a better-funded one can simply match you. Some of these pains are worse than fragile. They're the kind a single competitor or one new piece of infrastructure can erase outright, which means your whole differentiation has an expiry date you don't control and can't see.

An empty lane is a pain that's real but that few or no competitors run hard at. There's room to stand out because nobody else is standing there, and it resists fast copying, because copying means building a capability the incumbent doesn't currently have rather than nudging a number on a pricing page. The catch is that empty lanes are harder. They're usually empty because the pain is softer to articulate, harder to measure, or genuinely difficult to solve. You're trading "easy to enter, easy to lose" for "hard to enter, hard to lose."

I picked the empty lane on purpose, and here's the tell I trust. The crowded lanes were the comfortable choice. The comfort was the warning, not the reassurance.

## I paid the measurement tax knowingly

The empty-lane pain came with an immediate bill, and I want to be straight that I walked into it with my eyes open rather than tripping over it later.

The crowded-lane pains were all easy to quantify. A pain measured in money or seconds hands you a number on day one. The empty-lane pain I chose was softer, the kind customers feel sharply but that resists a clean metric, and it resists hardest at the start, when the product has no usage to measure in the first place. Choosing it meant signing up for a harder measurement problem than any of the alternatives would have handed me.

The mistake would have been to let measurability drive the choice. A meaningful pain you can't yet measure perfectly beats a shallow pain you can measure exactly, as long as you actually deal with the measurement problem instead of pretending it isn't there. So I dealt with it in two moves. First, the headline metric for the product is deliberately not the metric for the differentiating pain. The headline is something that shows a pulse from the very first working transaction, so the product looks alive from day one even before the harder pain produces any signal. Second, the differentiating pain gets its own driver metric, a proxy for "did we actually take this pain away," tracked next to the headline rather than as it. That split lets the product show early life without me lying to myself that the soft pain is easy to read.

## Nobody publishes how thin their evidence is, so here's mine

This is the section that matters most and shows up least in product writeups, so it gets the most honesty.

The evidence I locked this decision against was real, and it was also weak in specific ways I can name. I wrote those weaknesses down next to the decision instead of burying them, because a decision is only as trustworthy as the worst thing you're willing to admit about it.

The evidence was public competitor reviews, and public reviews lie in a consistent direction. People write them when something goes wrong and say nothing when things work. So the loudest pain in a pile of reviews is the most emotionally charged failure, not the most common experience. The data told me which failures hurt most when they hit. It could not tell me how often they hit. "Loudest" and "most frequent" are different questions, and review data physically cannot separate them. That one limitation inflates the apparent size of the pain I chose, and I locked the decision while staring straight at it.

The sample sizes were uneven, some large and some small, which means my confident-sounding reads on the small-pool players were weakly grounded and I knew it. The pattern grades I assigned, this pain is dominant, that one is minor, look quantitative on the page. They were a judgment call from reading review weight, not a counted frequency. Directional, not measured. I labeled them that way rather than letting a tidy-looking table imply more rigor than it had earned.

And here's the admission I least wanted to write. I never spoke to a single real customer in my actual segment before locking this. The entire pain is inferred from competitor reviews, not heard from the person I'm building for. Reviews told me what the incumbents do badly. They did not tell me what my customer ranks first, and those are not the same thing. The gap between them is the largest risk this decision carries, and it's the one I have the least cover for.

I locked it anyway. You cannot wait for perfect evidence, and a directional read against real signal still beats a confident guess against nothing. But locking it with the weaknesses written down means the next person, including the version of me who picks this up in three months, knows exactly which assumptions are holding the weight and which ones are unproven. That's the line between a decision and a hunch wearing a decision's clothes.

## The way I'm most likely to be wrong

One more piece of honesty, because this is the failure mode I rate highest and it isn't the obvious one.

Sometimes the reason incumbents are bad at a pain isn't that they're neglecting it. It's that they're bound by a constraint, and if I'm entering the same market, that constraint binds me too. There's a real version of the future where I discover the thing customers complain about is downstream of a rule I also have to follow, and the best I can do is handle it more gracefully than the incumbents, not make it disappear. If the win shrinks from "we don't have this problem" to "we explain this problem better," the wedge is a lot thinner than it looked the day I chose it.

I'm carrying that as a live risk, not a solved one, with a flat condition for reversal written next to it. If the constraint turns out to bind me as hard as it binds them, the differentiation may be too thin to hold a product up, and the wedge needs choosing again. Naming the thing that would make me reverse is, I'm now convinced, the single most useful line on the whole record. A decision you can't describe the undoing of isn't a decision. It's a belief you haven't noticed you're holding.

## What I'd do differently

I'd get real customer signal before locking, not after. The one soft spot running through this whole method is that I let competitor reviews stand in for the customer's own voice. They're a fine place to start and a bad place to stop. A handful of real conversations, asking the customer to rank the pains themselves instead of inferring the ranking from how loudly competitors get reviewed, would either have confirmed the choice on solid ground or caught a mis-ranking while it was still cheap to fix.

I'd separate loud from frequent earlier. I knew review bias existed, and I still built nothing to correct for it. Even a rough pass, weighting by review volume, or hunting for the quiet pains that show up in passing rather than in outrage, would have given me a more honest picture than the raw loudness of the pile.

I'd write the reversal conditions first, not last. The conditions that would make me undo this ended up at the bottom of the record, as a reflective afterthought. They're the most useful part of the whole thing, and they belonged at the top, as a live list of what to go validate, not a footnote I reached at the end.

## What it comes down to

Don't write the core-problem sentence from intuition in a kickoff room. Park it, name the customer, and force the pain to come from evidence. When the evidence lands, choose the pain that both hurts most and is least solved, which almost always means the harder empty lane over the comfortable crowded one. Pay the measurement tax the meaningful pain charges you instead of retreating to a shallow pain you can measure on day one. And write down, right next to the decision, exactly how thin your evidence is and what would make you walk it back.

The method doesn't make the decision safe, and I want to end on that rather than pretend otherwise. The pain I chose still rests on inferred evidence I've labeled as thin, and it may yet turn out to be a constraint I can't beat rather than a gap I can fill. What the method buys me is not safety. It's legibility. Every assumption this decision stands on is written where the next person can find it and test it. A wrong decision you can audit is recoverable. A right decision you can't explain is luck, and luck doesn't survive contact with the next decision that leans on it.
