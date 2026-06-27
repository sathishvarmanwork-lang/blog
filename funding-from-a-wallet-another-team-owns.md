# Why I funded a new payments product from a wallet another team owns, and how I kept the integration down to a single debit-and-refund seam

I was building the business version of a cross-border payment product on top of an engine that already ran the consumer version. The engine did almost everything the new product needed. It held the recipient details, worked out the exchange rate and the fee, checked each payment for sanctions and money laundering, handed the payout to an overseas partner, and tracked the whole thing from sent to received. One job in that pipeline did not carry over from consumer to business, and it was the first one. Where the money comes from.

In the consumer flow the customer pays from their own wallet balance. The last step of the send is, more or less, pay from wallet balance. A business is not a person with a personal wallet. So the open question was short to write and large to answer. Where does the money come from for a business payment, and how much new machinery do I build to hold it.

The decision looked like a question about wallets. It was really a build-or-borrow call, and I want to be plain that I treated it as one. I chose to borrow a business wallet another team already ran instead of building a funding store of my own. That choice was cheap in code and expensive in a way no diagram shows. It put the product's funding on another team's roadmap. I made the call anyway, and I wrote the dependency down as the decision instead of hiding it under the word reuse.

## The funding question was really build or borrow

The framing that mattered was not which wallet, it was build it or borrow it. Once I saw it that way, the options sorted into one I would own end to end and three ways of leaning on something that already existed.

Building my own business funding store felt like control and was the worst trade on the table. It is the most code, the most new compliance surface, and the most net-new logic on the critical path, all to rebuild a business wallet the company already ran somewhere else. Pure cost, no edge. Nobody picks a payments product because its internal balance ledger is hand-built. I would have spent my scarcest early effort rebuilding a solved thing.

Reusing the personal consumer wallet, the one the existing flow already pays from, was the laziest borrow and it broke the moment I looked at it. A company's money is not one employee's personal balance under a personal limit. And it falls apart as soon as a business has more than one person allowed to move money, which is most businesses worth having. The existing flow fit a single human sender. A business is not that.

Pulling money straight from the business bank account on each payment, with no held balance, sounded clean and hides its cost in timing. It means a new bank-rail integration, settlement timing I do not control sitting in the middle of a send, and no money parked ahead of time for the step where the engine has to set funds aside with the overseas partner before the payout clears. A held balance makes that step easy. Pull-from-bank turns it into a timing problem on every transfer.

The fourth option was to borrow the business wallet another team already ran. It came with top-up already built, a real balance, and a second-person approval feature I would otherwise write myself. I took it. The rule was the same one that ran the whole first slice. Reuse what exists and add the smallest new piece, so the pilot is not held up waiting on something I have to build before I can prove anything.

## The point was to leave the send engine alone

Borrowing the existing wallet only pays off if the send engine stays exactly as it is. I had already decided to build on that engine rather than its half-finished replacement, and that bet only works if funding logic does not bleed into it. So the whole integration comes down to two touchpoints and nothing else.

When a payment is sent, take the amount and the fee from the business wallet, then let the engine do what it already does. If the payment fails, put the money back. That is the seam. The engine does not learn what a business wallet is. It learns that something hands it funded value at send time and takes a refund instruction back on failure, which is the same shape the consumer wallet already had. The funding source changed. The pipeline did not.

That is the whole appeal of the choice, and saying it plainly is also how I caught what was wrong with it. When the integration is one debit and one refund, it is tempting to call it done. The seam is clean on the happy path. The seam is where all the risk lives on every other path.

## Borrowing moved the risk onto a roadmap I do not own

Here is the part that does not show up when you draw the boxes and arrows. The wallet I chose to borrow belongs to another team, and the exact thing I need it to do, pay any overseas supplier from a business balance, is not something it does yet. Today that wallet pays merchants in known flows. Paying a third party abroad is on that team's plan, not shipped. So my funding model is a recommendation that depends on their go-live and on them letting my use case ride on it.

I do not own that date. That is the whole sentence. Building my own funding store would have been more work and fewer unknowns, because the unknowns would have been mine to close by writing code. Borrowing turned a build risk into a coordination risk, and a coordination risk is the kind you cannot close at your own desk. You close it when another team ships something on their schedule.

This is the trade people skip when they say reuse like it is free. Reuse is usually right. It was right here. But it is not free, and the cost is paid in the one thing engineering plans rarely price, which is waiting on another team's order of work. I would rather name that out loud and carry it as a known risk than let the word reuse slip it past review.

## The one-line swap hides a real distributed transaction

The second thing the clean seam was hiding sits closer to home. Swap the funding source is one line in a product document. Set funds aside, take them, send them, and refund them on failure, across two systems that do not share a database, is a distributed transaction, and the failure cases are the actual work.

What happens when the wallet debit succeeds and the payout to the partner times out. What happens when a refund fires, then fires again because a retry did not know the first one landed. What happens when money is set aside but the send never completes and nobody puts it back. The happy path is the line I wrote. Everything that makes this hard lives in the paths I did not, and a document written at the product level makes the whole thing look cheaper than the engineering will turn out to be. I flagged that in the record. I also know a flag is doing a lot of work that a real interface has to finish.

## How I am most likely to be wrong

The failure I rate highest is not a bug. It is the dependency coming back with a no, or a not yet that lands after the pilot needs it. If the other team confirms the business wallet cannot pay an overseas third party inside my window, my clean seam has nothing on the other side of it. The fallback is the bank-pull option or a purpose-built float, both of which I turned down for being more expensive, so the reversal is not cheap. I am carrying that as a live risk with a flat condition next to it. If that capability is not coming in time, the funding model gets chosen again from the options I had already rejected.

The second way I am wrong is quieter, and it attacks the reason I made the choice at all. If the wallet's real debit and refund behaviour does not offer a clean set-aside-and-release step, then the engine I worked so hard to leave alone has to start carrying funding state anyway. The moment that happens, the cleanliness I was protecting is gone, and the borrow has quietly turned into a build with extra steps and a worse owner boundary. The thing that justified reusing the engine and the thing that justified reusing the wallet are the same thing, a clean seam, and a bad step on the wallet side breaks both at once.

## What I would do differently

Pin the go-live and the use-case approval before writing the send flow as if the wallet is a given. I wrote the flow assuming the capability and then noted the dependency under it. That is backwards. The dependency is the riskiest part of the decision and it belonged at the top, as the first thing to confirm, not as a footnote under a flow that already spends it.

Spell out the debit-and-refund contract at the engineering level before calling it a seam. Name the set-aside step, the timeout behaviour, and the key that stops a refund from running twice. A seam you have only described in prose is a seam you are hoping exists. The difference between hoping and knowing is exactly the interface I have not written down yet.

And I would not have tied the second-person approval I get for free to that same unshipped capability without a fallback. Getting maker and checker approval thrown in by the wallet is a real win. Making it depend on the one thing I do not control means a win I cannot actually promise the pilot.

## What it comes down to

The thing I keep coming back to is that the cheapest option and the riskiest option were the same option. Borrowing a wallet I did not build saved me a funding stack and handed me a dependency I cannot schedule. That trade is usually worth taking and I would take it again here. But reuse is the most overloaded word in an architecture review, and most of the time it is standing in front of a sentence the diagram does not show. Someone else owns the thing you are leaning on, and their roadmap just became part of yours.

The honest version of this decision is not "I reused the business wallet." It is "I bet the product's funding on another team shipping a capability they have not shipped, and I judged that better than building it myself." Write that sentence where the next person can find it and the decision can be checked, including the day it needs undoing. Leave it at reuse and you have shipped a dependency nobody chose on purpose.
