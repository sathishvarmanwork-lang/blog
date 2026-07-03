# Why I built the business-verification step on another team's records instead of a live registry integration, and how I now treat every reused field as unverified until I check where it came from

Before a business can move money through the product, it has to be verified. The company name, the registration number, the registered and trading address, the nature of the business, the directors, the shareholders, the real people who ultimately own it, and proof the company exists. This is know-your-business, KYB, the company-level form of the identity check a bank runs on a person. It is a regulatory gate, not a feature. No verified business, no payment.

So the first design question was never whether to collect this data. It was an architecture question about sourcing. For each field, where is the authoritative source, and how much new machinery do I build to move the value from that source into my system. That question, asked field by field, is a build-versus-buy call wearing an onboarding costume.

There is an authoritative source for a large part of the set. Every company is registered with the national company registry, and that registry is the system of record for the public facts, the name, the number, the address, the nature of business. System of record means the one place whose copy is treated as true, and everyone else has to reconcile back to it. The clean architecture is obvious. Integrate with the registry, read the authoritative fields on demand, and never depend on a stale copy.

I did not build that integration for the first version. The reason I declined it is the ordinary half of the story. The half worth writing down is what one field taught me about the word reuse.

## The cheap source was a copy, not the source

Another product in the same company had already onboarded many of these same businesses once before, for its own purpose, and when it did, it read the public company facts from the registry and stored them. So the authoritative fields already sat in a datastore next door, captured and owned by another team. I could read that datastore. Integrating with the registry itself meant getting access granted, mapping an external schema, and owning the registry's downtime as my own failure mode. For a first version that verifies a single pilot business, that is a large build and a fresh external dependency, all to automate a read I perform exactly once.

So I did the thing that looks free. I classified every KYB field into two sets. One set was the public facts the neighbour already held. The other was everything it did not hold. Then I read the first set from the neighbour and had a person type the second set in by hand.

| Field | Decision | Note |
|---|---|---|
| Registration number | Reuse | public registry particular, held by the neighbour |
| Trading address | Reuse | held by the neighbour |
| Nature of business | Reuse | held by the neighbour |
| Name, legal form, proof of existence | Build (partial) | name held, add legal form and proof documents |
| Registered address | Disputed | one requirement sheet says held, one says gap |
| Directors, shareholders, beneficial owners | Build | full identity and ownership share, by hand |
| Signatories and senior management | Build | by hand |
| Authorised person to act | Build | by hand |

Reuse here does not mean live. The neighbour did not hold a live line to the registry either. It held a copy taken at the moment it onboarded that business, which could be years old. So the fields I marked reuse are two hops from the truth. The registry wrote them once, the neighbour copied them once, and I now read that second-hand copy with no link back to either write. I did not shorten the distance to the source. I inherited someone else's distance to it.

## Reusing a field makes another team's datastore my system of record

This is the part the neat two-column table hides. The moment I read a field from the neighbour's store instead of the registry, that store becomes my system of record for that field, and I have taken a hard read dependency on data whose write path I do not own. I cannot see when it was captured. I cannot trigger a refresh. I get no event when the underlying company changes its address or its directors or its ownership. The coupling is invisible on an architecture diagram, because on the diagram it is only an arrow labelled reuse, but it is real coupling all the same, and it runs to a team whose roadmap and data model are not mine to steer.

I priced the integration I skipped. I did not price the data I inherited. Those are different bills, and the second one comes due later, quietly, at compliance review rather than at build time.

## One field with two contradictory answers

Here is what turned the abstract risk into something I could point at. Two internal requirement sheets, describing the same onboarding, disagreed about a single field, the registered address. One sheet marked it a gap still to collect. The other marked it already held by the neighbour. Two documents inside the same programme, describing one field, could not agree on whether the data even existed.

That contradiction is the whole record in miniature. The reuse label is a claim about what another team's system contains. For this field, nobody could confirm the claim. On the table it looked settled, a tidy entry in a column. Underneath, it was an unverified assumption about someone else's data, written down in the same confident ink as the facts I had actually checked. And the lesson does not stay on that one field. If two sheets I can see contradict each other on the address, I have to assume the same unchecked doubt sits under every field only one sheet ever described.

## The set I chose to key by hand is the compliance-sensitive one

Look at which fields fell into manual entry. Not the harmless ones. The beneficial owners, the signatories, the proof the company is real. These are the fields a regulator cares most about, the ones where a wrong value is not a cosmetic bug but a compliance failure, and I put them on a person typing into a form. Hand-keyed data on the most sensitive part of the record carries both typo risk and fraud risk, and it is exactly the surface a registry integration would have hardened with authoritative reads. For one pilot business under a careful reviewer, manual entry holds. As the standing design for the sensitive fields, it is the weakest link sitting on the highest-value data.

## The decision has a shelf life, and volume sets the clock

This is the right call for one business and the wrong call for a thousand. Declining the integration is correct while the only thing it would buy is automating a read I do once, and while it would otherwise sit on the critical path to getting started. It becomes wrong the moment onboarding volume climbs. Then the manual set is the throughput bottleneck, and the staleness I accepted on the reused set stops being a theoretical risk and becomes a real payment failing its compliance check because the address on file moved two years ago.

So I did not cancel the integration. I deferred it, and I wrote down the trigger that brings it back, which is the pilot ceasing to be a pilot. A deferred dependency with a named trigger is an honest piece of architecture. A deferred dependency you quietly hope never returns is a debt you forgot to log.

## What it comes down to

Reuse is the most comfortable word in an architecture review and the one that hides the most work. Reading the neighbour's data was the right call and I would make it again. But the neat reuse column was a set of promises about data I do not own, and one of those promises fell apart the moment two documents sat side by side.

So the discipline I hold now is small and it is not optional. Before I write reuse against a field, I check where the value actually originates, whether the owning system truly holds it, and how old the stored value is. A field marked reuse is an assumption until someone has opened the source and looked. The one field I could not confirm is the field that told me to stop trusting the column and start auditing the data behind it.
