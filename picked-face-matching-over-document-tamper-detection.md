# Why I picked face matching over document tamper detection, and how I now check whether the training data exists before I decide anything else

Someone opens two pictures side by side. One is a selfie. The other is the photo printed on a national ID. They look at both faces, decide whether it is the same person, and move on to the next pair. Later they open a work visa and look for signs that someone has edited it. A date sitting slightly off the line, or a photo whose edges look too clean. They do this by eye, one submission at a time.

Those two checks were handed to me as the two most painful manual jobs in the flow. Both of them hold a person up until a reviewer clears them. I had capacity to build one system, not two.

I picked face matching, including the part where I think my own reasoning might have been bent toward the answer I wanted.

## Two problems that looked like siblings

On the surface these are the same job. Both take an image, ask a hard visual question about it, and today the answer comes from a person squinting at a screen. Both are the kind of work that people describe as slow and draining. If you sketched them on a whiteboard they would sit next to each other under a heading about identity checks.

They are not siblings. They split on one thing, and it has nothing to do with how clever the model has to be.

Face matching asks whether two faces belong to the same person. That question has been worked on for a decade by people with far more compute than I have, and it is finished. Open models with permissive licences score around 99 percent on the standard test sets. The weights are free to download. I do not have to train anything. I have to call something.

Tamper detection asks whether a document has been edited after it was issued. That question is not finished, and more importantly, the thing I would need to answer it does not exist.

## Why one of those two cannot be built

There is no public dataset of real work visas, tampered or clean. I went looking for one specifically, and the answer came back the same from every direction. This kind of document does not circulate as training data, because privacy law and government restriction keep it from circulating. I am confident about this one. It is not a gap in my search.

The closest thing that does exist is IDNet, published as `cactuslab/IDNet-2025` under CC-BY-4.0 and described in arXiv:2408.01690. It holds 837,060 images across roughly 490 GB, covering 20 document types from 10 US states and 10 European countries. Driver's licences, national ID cards, passports. The tamper types are inpaint-and-rewrite and crop-and-replace.

It contains no visas. It contains no work IDs.

I surveyed the rest of what is out there and got the same result. DocXPand, SIDTD and FMIDV all hold synthetic forged identity documents. MIDV-2020 holds clean documents you can tamper yourself. FantasyID exists to give you something commercially safe to evaluate against. Not one of them covers visas. On the model side there is real published work, including ManTraNet, CAT-Net, TruFor, FakeShield, MVSS-Net and PSCC-Net, but most of it is research code rather than something you drop into a service.

The constraint here is supply, not difficulty. What decides whether I can build something with AI is whether the training data exists, and for this problem it does not.

## A shortcut I looked at seriously and turned down

The obvious move is to train on IDNet anyway and point the result at visas. Documents are documents. Editing leaves traces. Surely it transfers.

It half transfers, and the half that fails is the half I need.

A tamper model learns two separable things at the same time. The first is the generic signature of editing itself, meaning smudged edges, odd noise patterns, seams where something was pasted in. That part genuinely carries across document types, because a copy-paste seam looks like a copy-paste seam wherever it lands. The second is what each document type looks like when it has not been touched. That part does not carry at all. A work visa and a Texas driver's licence share almost nothing visually. A model trained to know what "normal" looks like on one has no idea what normal looks like on the other.

Which means the most I could say at the end of that build is that the model scores 94 percent on identity documents and has never been tested on a visa. That claim is worthless to the people who asked me the question, because they asked about visas. Shipping an unmeasured model into a check that decides whether a real person gets through is the kind of risk I am not willing to take on a first build.

I also spent time on the idea of making the data myself. Design plausible visa layouts, generate a large batch, apply known edits to a portion of them, and train on that. It is a real option and people do it. I turned it down because of what the model would actually learn. It would learn the marks left by my own generator, not the marks a person leaves with a photo editor and a deadline. I would have no way of telling those two apart from the inside, and I would have burned several weeks to arrive at a number I could not trust. That is worse than not building it, because a number you cannot trust still gets quoted.

## Proving it works before it touches a real person

The second filter I applied was whether I can measure the thing before it goes anywhere near a real decision. This matters more than it first looks, because both of these checks sit directly in the path of a person waiting to be let in, and the two ways of being wrong are not equally bad.

If the system wrongly clears someone who is not who they say they are, that is a fraud failure and a compliance failure. If the system wrongly blocks a genuine person, that is a different kind of loss, and that person often does not come back to try again, though I should say I am assuming that rather than reporting it. Any cutoff I choose is a trade between those two, and I cannot choose it sensibly without a curve showing how the system behaves as conditions get worse.

For face matching I can build that curve. I can assemble my own test set, deliberately degrade the images with glare, motion blur and low light, and plot exactly where accuracy starts to fall away. That chart is the actual deliverable. It is what lets someone pick a threshold on purpose rather than by feel, and it is what tells them which slice of cases still needs a human.

For visa tampering I cannot build that curve, because building it needs a visa test set, and I have already established that one does not exist. I would be handing over a system whose failure behaviour is unknown, attached to a check that has legal weight.

There was a third filter, and it only mattered because the first two had already decided the outcome. My own engineering charter aims at the band above the waterline, meaning building products that call pre-trained models. Fine-tuning and training sit below that line and are marked for later. The visa project is a training project almost end to end. The face project is a product with a model call sitting inside it. That is the work I want to be getting better at.

## What I gave up by choosing this way

I want to be plain about the cost, because the choice was not free.

Tamper detection is arguably worth more than the one I picked. It catches someone deliberately trying to get through with an altered document. Face mismatch, most of the time, catches ordinary error instead, like a bad photo or the wrong file uploaded. I should flag that this split between deliberate fraud and ordinary mistake is my own assumption. Nobody measured it for me and I did not verify it. Taking it at face value, I chose the problem I could build over the problem that was worth more, and I did it with my eyes open.

Face matching is also unglamorous. There is nothing new in it. The model is a commodity and anyone can download the same weights I did. All of the value lives in the system wrapped around the model, which means the image quality gate in front of it, the choice of cutoff, the rules for escalating an uncertain case, and the measurement that justifies both. If I under-build that part, the whole pivot was pointless and I will have shipped a wrapper around a download.

And deferring the visa work leaves that gap exactly where it was. People still inspect those documents by eye. Nothing about my decision improves that, and I should not pretend otherwise.

I did look at buying rather than building. Jumio, Onfido, Au10tix, Regula and Sumsub all sell document and identity verification, and for most organisations comparing those vendors is the right first step before writing any code at all. I ruled it out because this is also a project I am building to sharpen my own engineering, and buying teaches me nothing. That is a personal reason, not a general one, and I would give a company different advice.

## What I am building instead

The design is five steps and I am building it now.

An upload service takes the two images. A quality gate checks whether the photos are good enough to judge at all, and rejects them early with a reason rather than passing garbage down the line. The face match runs against a ready-made model. A decision step turns the score into one of three outcomes, meaning match, unsure, or reject. An evaluation harness measures the whole chain under degraded conditions and produces the curve I described earlier.

For the model itself I am using AuraFace-v1, which ships under Apache-2.0 and reports 99.65 on LFW and 95.19 on CFP-FP on its published model card. CFP-FP is the benchmark that varies head pose, which is the one that matches this situation, because an ID photo is flat and front-facing while a selfie is taken at whatever angle the person happened to hold the phone. Those figures come from the people who published the model and I have not reproduced them myself.

The design point that makes this safe to put in front of a real check is that it sorts the queue for the reviewer instead of replacing them. Clear matches and clear rejects go through on their own. Anything in the uncertain band still reaches a person, and it arrives with the score and the reason attached so the reviewer starts from something rather than from nothing. A system that removes half the queue and hands the hard half over in better shape is worth more than a system that claims the whole queue and quietly gets some of it wrong.

## A pattern in my own reasoning that bothers me

Every filter I applied pointed the same way. Buildability, provability and charter fit all favoured face matching. When all of your criteria happen to agree with the answer that is also the least work, you should be suspicious of your criteria.

My defence is that the first filter is not a matter of taste. The visa data either exists or it does not, and it does not. That is a binary fact I checked against the primary source rather than a preference I dressed up as one. If a licence-clean visa dataset appeared tomorrow, the first filter flips and the whole argument has to be reopened.

But I will not pretend the rest of it was that clean. The second and third filters are both arguments I constructed, and I constructed them after I already had a feel for where this was going.

There is a bigger weakness underneath all of it. I decided this on qualitative input. I was told both jobs were painful and slow, and that is genuinely all I had. I do not know the daily submission count, the average time a reviewer spends on one, the backlog size, the current error rate, or what either problem costs. With those numbers this would have been arithmetic. Without them it was an argument, and an argument is easier to bend toward the answer you already like.

I should have asked for the numbers before deciding rather than after. That is the thing I would change if I ran this again, along with putting a vendor comparison ahead of any build decision at all.

## What I still cannot answer

The number I want most is one nobody has, which is how accurate the people doing this manually are today.

Without that baseline I cannot make the only claim that actually matters. I can tell you the model scores 99.65 on a benchmark and I can show you my own curve under glare and blur. I cannot tell you whether it beats a careful reviewer on a Tuesday afternoon halfway through a long queue. If it is worse, then the system removes work and adds risk at the same time, and the people who trusted me with the decision end up worse off than before.

That baseline is a piece of measurement work. It needs a sample of past cases, a second opinion on each of them, and somebody's time.

It is also the first thing I should build.
