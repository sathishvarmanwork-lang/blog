# Why I built interim payment funding on a ledger I own instead of a connected wallet, and how I audit a dependency's spec before it becomes load-bearing

I am building a cross-border payment product. The architecture needed a funding source, and a connected wallet owned by another team seemed like the right one. I wrote two sections of the design around that before I read their spec.

When I read it, I found the wallet cannot do what I needed. That sent me back to the existing payments engine, which had the ledger and hold machinery I needed once I looked properly. The funding path in the current design runs on a balance inside the engine I own.

## An assumption I did not check

The product moves money from a funded account to a recipient outside the platform. The business has to fund that payment from somewhere, and the design needs to say exactly where.

There is a wallet product, built by a connected team, that businesses use to hold and move money inside the same platform. On the surface it looked like the right source. Businesses already put money there. The product operates on the same platform. The debit model would be simple. When the business initiates a payment, the system debits the wallet. Two product lines, one debit, no new balance for the user to manage.

In a connected platform, this kind of assumption is easy to make. Teams share infrastructure, share users, and share the concept of a balance. When you are designing a payment that needs money to come from somewhere, and there is already a place where users put money, the instinct is to start there. That instinct is not wrong in principle. It is wrong when you follow it without checking what the other product actually supports. Each team has its own scope, its own use cases, and its own timeline. Shared infrastructure does not mean shared capabilities.

I wired the assumption into the design. My funding section said the payment debits the connected wallet. Two other sections of the architecture referenced the same assumption. I moved on.

I did not open the connected team's product document before writing any of this.

## What I found in their spec

When I opened the wallet team's document and went through their payment flow section by section, I found three concrete problems.

The wallet supports bill payment, offline payment, and online cashier payment. The payment type I was building for is not in the list. The wallet was built for specific merchant and consumer use cases, and overseas remittance is not one of them.

The execution model is also architecturally separate from anything I can call into. When a business pays from the wallet, the wallet product runs its own internal gate. The business enters a PIN, the system runs a risk check, checks the transaction limit, then debits the wallet balance. That gate is a security boundary. It protects the wallet's funds and authorises the user's action. My payments engine cannot reach in and trigger it. There is no payment interface I can call as an external system. For my product to debit the wallet, the connected team would need to build remittance as a new supported use case inside their gate, with the appropriate risk and limit rules attached. That is their work to design and build, not a configuration I can add from my side.

The wallet also has a per-payment cap that would block most of the payments I was designing for. A typical payment in my use case would exceed it.

Three problems, all from one document I had not read. The wallet is built well for what it does. It just does not do what I assumed.

## Why neither obvious fix worked

The first path was to keep the wallet as the dependency and wait for the connected team to add remittance support. They would build it as a new use case inside their gate, expose an interface my engine could call, and raise the per-payment cap. The design stays as-is.

The connected team has no committed date for any of this. Their roadmap has other priorities, and remittance is not on it today. Keeping the dependency means my launch timeline is determined by a backlog I do not own and cannot commit. A dependency with no date is not a dependency. It is a block dressed up as a plan.

The second path was to build a new funding mechanism from scratch. A new balance store, a deposit flow, hold and debit routines, a refund path for cancelled payments. No dependency on anyone.

The existing payments engine already has most of this. It has a ledger. It has routines that place a hold on funds when a payment is initiated, keeping them reserved while the payment is in progress. It has the debit and refund flows that settle or reverse the hold when the payment completes or is cancelled. Building a parallel mechanism duplicates what already works and creates a second codebase to maintain indefinitely.

## Moving the funding path

I moved the funding path onto the existing payments engine.

The product funds and debits payments through a balance the product controls inside the engine, not through the connected wallet. The business tops up this balance. When a payment is initiated, the engine places a hold on the required amount. When the payment settles, the engine debits the hold. If the payment fails or is cancelled, the engine releases the hold and returns the funds.

The ledger, the hold flow, and the debit and refund routines already exist in the engine. They handle real transactions today. The new parts are the top-up mechanism and the business-facing balance state for a remittance use case. The engine extends to support a new balance type, following the same ledger pattern it already uses, with the specific hold periods, compliance flags, and refund rules that remittance requires. It does not need a parallel replacement built next to it.

The connected wallet stays in the longer-term picture. When the connected team builds remittance as a supported use case inside their gate and raises the per-payment cap, the migration is one change at one point in the payment flow. The funding debit moves from the internal balance to the connected wallet. The rest of the architecture does not move with it. Replacing the funding mechanism later is a contained migration, not a rebuild.

## A tradeoff I accepted

Two builds instead of one. The interim balance now, and the migration when the connected team is ready. That is real extra work with a date I cannot predict.

I accepted it because the alternative was worse. Tying the launch to an undefined timeline on another team's roadmap is not a safer position. It is the same failure with a longer fuse and the cause sitting outside my control.

I could confirm the engine has the ledger and hold machinery by reading the codebase. What I had not yet resolved with my team is the exact form of the interim balance. A prefunded account and a collection account are different mechanisms with different reconciliation models, and that is something my team needs to work through together before it goes into the implementation. I logged it as an open question in the design document. It is not invented, not assumed, and not locked before the team works through it. What the open question asks for specifically is the technical design for the interim balance and a timeline from the connected team for when they will be ready to take remittance on. Until both of those exist, the open question stays open and the interim balance is described at the product level only, not the implementation level.

## A correction mid-build

There is a distinction that matters here, and I got it wrong on the first pass.

When I first worked through this decision, I described the existing engine as already having a funded balance ready for business remittance. That was too loose. The engine has the ledger and the hold and debit machinery. It does not have a business remittance balance ready to draw from today. The engine's ledger has historically been used for a different payment type. For a business sending money overseas, the balance type, the hold variant for a remittance use case, and the associated reconciliation all need to be built as extensions of what already ships.

The difference between "reuse" and "extend what is shipped" sounds like a framing choice. It is not. Reuse means I point to an existing thing and declare it done. Extend means my team has real new work to plan for, with a scope to estimate, test cases to write, and edge conditions to handle. A design that says "reuse" when the correct word is "extend" looks complete on the surface and hides a build underneath. Getting the framing wrong would have produced exactly that. A design that appeared ready but was not.

I caught it while working through the decision and corrected it before it went into the design document. The document now says "extend", and the open question captures what the team still needs to design.

## What I check before writing a dependency

The failure here was a reading failure. I wrote a connected product into the architecture as a dependency before reading the document that describes what it actually does. That is the specific thing to change.

What I do now before I write a connected product into a design is open their product document and read it. Not a summary. Not what someone on their team told me it does. The actual document. Specifically, I check whether the use case is supported, how the integration model works, what the transaction limits are, and whether the capability I need is live today or only planned.

The fourth check is the one I would have missed before. A feature that is planned but not yet built is not a dependency. It is a future dependency with an unknown date. It needs to be treated as an open question in the design, not a settled line in the funding section. The only thing worse than a wrong assumption is a wrong assumption that looks like a decision.

If any of the four checks is unclear, I ask the connected team directly before writing the dependency into the design. I do not assume. And I pay attention to what the document does not say, not just what it does. An omission (no remittance use case, no API section) is as informative as the content that is there.

This check takes under an hour when done before the design is written. The rework, when the assumption breaks after weeks of building on it, is measured in days.

Put more sharply, a line in my design that says "the system debits the wallet" is a factual claim about another team's product. If I have not read their document, I have no basis for that claim. I have written a sentence that looks like a design decision but is actually an unchecked assumption.

## Still open

The connected team has no committed date for remittance support. The design has an open question asking for that timeline and defines the migration plan that starts when the date exists. Until then, the interim balance is the funding path.

Whether the interim balance runs for three months or three years depends on the connected team's roadmap. That is a genuine unknown. The product design does not pretend otherwise.

The funding path no longer depends on an unfinished capability owned by another team, on a timeline I cannot control. The design can move forward. The migration happens when the connected team is ready, and the architecture makes that migration a single swap at one point in the payment flow, not a rebuild. Until that point, the interim balance does the job, and my team knows exactly where the migration lands when the time comes.
