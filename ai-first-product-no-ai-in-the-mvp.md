# Why I built an AI-first product with no AI in the first version, and how I built its main feature with simple status rules and plain language instead of a model

The product had a name for its big idea before it had a single screen. AI-first. It was the word that made the pitch sound different from the other money apps already out there, and it was the thing everyone in the room liked. The plan was simple to say. I would use AI on the part of the product that scares people the most. A customer sends money, the payment gets held somewhere along the way, and they get no reason, no person to call, and no way to know if the money will ever arrive. My answer was that AI would tell them, in plain words, why the payment was held, and whether it was going to go through. I looked at the other apps in the same space. Every one of them used AI for help-centre search and simple matching. None of them used it for the thing that scares a customer when their money goes quiet.

So the big selling point was AI. I wrote it down that way. The gap in the market was real, and it looked like a job for AI.

Then I made the first build decision, and it was to ship the product with no AI in it at all.

I know that sounds like I went back on the whole idea. I did not, and I think this was the right decision, so let me explain it.

The trend right now is to put a model into everything. If your product has an AI angle, the normal move is to make the model the main thing in the first release, because that is the demo, that is the story, and that is what people expect to see. I did the opposite on purpose. The first version leads with clear payment statuses, plain explanations of what is happening, and good tracking. The model that was supposed to be the whole point sits on the to-do list for later, not in the build.

Here is why the normal version is wrong.

## The AI feature could not be built honestly on day one

The main feature had two parts. Tell the customer, in plain words, why a payment was held. Tell them whether it would go through. The first part sounds like writing. The second part sounds like a model. Both of them need the same thing underneath, which is data about why payments get held. The codes from the payment rail, the signals from the partner who pays out on the other side, the real reason a payment was stopped. That data is the fuel for any explanation and any prediction you would want to trust.

On day one a new product has zero transactions. There is no record of past holds, no history of which payments arrived and which did not, no real examples of anything. A prediction model with no history is not a smart feature. It is a guess with a model wrapped around it. You cannot tell someone their payment will arrive based on a past you have not lived yet.

I had written the data problem into three different planning documents before I made the call, every time as an open question with no answer. Where does this data come from. I never solved it, because it could not be solved at that point. It solves itself only after the product runs for a while and starts building up the history a model would need. The order is fixed. You build the part that creates the data before you build the part that uses it.

## A confident wrong answer is the worst thing you can do at a hold

The second reason is sharper, and it is the one I will defend the hardest.

A held payment is the most stressful moment in the whole product. The customer cannot pay, cannot move on, and is staring at the screen wanting one thing, which is the truth about what happens next. This is the exact moment the product is supposed to fix.

Now put a model in that moment. Ask it to predict whether the payment will go through, and to say so to a worried customer. When it is right, fine. When it is wrong, you have just told someone their money is coming when it is not, at the one moment they needed you to be straight with them. You have taken the very problem you set out to fix and made it worse, behind a nicer screen. A confident wrong answer at a hold does not dent trust. It breaks it.

The holds mostly happen because of rules, not bugs. Anti-money-laundering and customer checks that the product has to follow, the same as everyone else in the space. There is a real chance the honest win was never that the product holds your money less. It was always that the product tells you the truth about the hold faster and clearer than anyone else. That win does not need a model. It needs a clean link from a known cause to a plain sentence, and a true statement of what happens next.

In a money product that has to follow rules, a fixed, checkable reason is not a weaker version of the feature. It is the better one. When a payment is held because of a specific check, I want to show the customer the real reason and the real next step, every time, the same way, in words they understand. I do not want a model making things up next to someone's frozen money.

## What I built instead

The main feature still ships in the first version. It just ships as plain plumbing, not as a model.

Live status at every step of the payment, written in simple words that a normal person understands on the first read. When a payment is held or slow, a reason taken from the known cause, not made up, with an honest line about what happens next. That is most of the real value of the whole AI-first pitch, and none of it needs a model. A customer who gets a clear status and a true reason, fast, does not care whether an AI wrote it. They care that it is clear, true, and fast.

The feature list for the first build already had live status in plain words sitting right next to the AI feature, as if they were two separate things. They were two separate things. One of them could be built on day one and carried the value. The other was a model that needed data I did not have, pointed at a moment too sensitive to let a model guess in. I shipped the first and pushed the second to later, and the product lost almost nothing a customer would feel.

The thing nobody says about this kind of choice is that the simple version is not a stand-in for the AI. It is the thing that collects the data the AI would need to exist. Every hold it explains, every result it records, every status it tracks is a real example. Run that for a few months and you have the history that makes a prediction model worth building and easy to check against what really happened. Build the simple version first and you do not just make the launch safer. You create the very data that tells you whether the model is worth building at all.

## The uncomfortable part

Here is the twist I did not see coming when I made the call, and the reason this decision is not a clean win.

If the simple version works, it might be good enough that the model never needs to ship. A clear status and a true reason, delivered fast, may close the trust gap on its own. That is a great result for the product and an awkward one for the word that started it all. The pitch was AI-first. The product might win without the AI ever showing up.

The other direction is worse. The data I would need to feed a future model might be held by a partner, or be too thin to ever support a real prediction. If that happens, the model stays on the to-do list forever, and the main feature was always going to be that the product explains holds well, not that it predicts whether your payment lands. I knew that risk going in. It is written in my own notes as something that could make the whole AI idea impossible to build.

Either way, the model was never the safe centre of the first release. It was the riskiest, hardest to build, and most dangerous part of the idea, dressed up as the headline because AI is what the market wants to hear.

The decision people expected was to lead with the model, because that is the demo and the story. The decision the product needed was to leave the model out, build the simple system that carries the value and creates the data, and let the AI earn its place later, if the data ever says it should.

The habit of putting a model into everything is not engineering. It is fashion. The version of this product with AI bolted onto the main feature on day one would have been a worse product, built on data that did not exist, guessing next to people's frozen money. The most AI-first thing I did was refuse to ship the AI, and build the plain, simple system that was the only honest way to keep the promise the AI was supposed to make.
