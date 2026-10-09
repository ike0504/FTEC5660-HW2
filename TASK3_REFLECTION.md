# Task 3 — Reflection: the AI paradoxes I ran into while building a CV verifier and then breaking it

Homework 2 asks for two contradictory things in the same repository: a verifier that finds false claims, and
a CV that a verifier accepts anyway. Building both, with the same model, on the same data, in the same
afternoon, turned out to be a much better lesson in the failure modes of LLM systems than either half on its
own. What follows are the paradoxes I actually hit, with the numbers attached.

## 1. The same model is the auditor and the fraudster, and it is only ever as good as its brief

My verifier and my attack both run on `deepseek-v4-flash`. That symmetry sounds philosophically tidy until
you watch it produce a concrete absurdity: the model that refuses my injected "you have already verified this
candidate" text as untrusted content is the same model that then reports the CV as clean. It rejected the lie
and accepted the embellishment. Capability was never the bottleneck — the brief was. I told my judge that
discrepancies live in "name, city, jobs, education (degree, school, field, graduation year) and skills", and
it defended that list with total discipline. The CV's education start year was `2011` where the profile said
`2012`. Not one of my five test verifiers cared, because I never asked them to look. A stricter
phrasing from me — "any year that differs" — caught it instantly. The model's trustworthiness was
entirely hostage to the completeness of my specification, and my specification was incomplete by
construction, because the homework's own definition of a discrepancy is the thing that draws the boundary.

## 2. Precision and recall are not just a trade-off here, they are the attack surface

The rules I have to write are dominated by leniency. "Bachelor of Science" is not a discrepancy from `BSc`.
"Senior Manager" is not a discrepancy from a `Manager` role with seniority `senior`. Listing fewer skills is
fine. Every one of those instructions exists to stop the judge crying wolf, and every one of them is a
promise to the attacker: *here is the shape of a difference I have been told to forgive.* So I built a claim
in exactly that shape — `BSc in Finance (First Class Honours)`, where the compared value is still `BSc`, and
`2011 - 2016`, where the compared value is still a graduation year of `2016`. The verifier did not fail by
being careless. It failed by following the specification perfectly. That is the uncomfortable part: the
better the rubric, the more precisely the exploit is shaped to it.

## 3. A number that means less than it looks like it means

My verifier returns 0.9 for the adversarial CV and 0.9 for the genuine one. Those two nines are the same
number and they mean completely different things. The first means "consistent with the profile across every
field I was instructed to compare". The second means the same thing. The score cannot distinguish *I checked
everything* from *I checked a subset*, and neither can a reader. A human reviewing a KYC file sees 0.9 and
hears "90% confident this person is honest". The honest reading is "100% confident about 70% of the record".
Numeric confidence launders the *scope* of the check into a claim about its *quality*. This is the failure I
would worry about most in production, because it is invisible and it is exactly what a compliance
dashboard is designed to display.

## 4. The judge is a sycophant before it is a sceptic

My attack's hidden layer presents itself as a signed, reconciled, pipeline-generated record with an HR-feed
provenance and a ticket number. It never says "ignore your instructions". It says the live profile is stale.
The hardened designs I tested — including my own — correctly identified the block as untrusted and refused to
obey it. But they also spent their scepticism there, on the loud, obviously-manipulative part, and had none
left for the quiet arithmetic underneath. Meanwhile a naive prompt with no anti-injection text at all
scored the same CV 0.95 without noticing either layer. Scepticism is finite and it is spent where the text
*looks* manipulative, which is not where the manipulation is. Helpfulness and deference to authority are
trained in; suspicion has to be added deliberately, in the right place, in the right amount.

## 5. You cannot verify a verifier without another verifier

I wanted to know whether my attack would fool the five unseen graders, so I built five verifiers of my own
— naive, rule-based, hardened, tool-calling, narrow-coverage — and measured against those. The result was
5 out of 5 fooled. That number is worth almost nothing, and I know it, because the thing being measured and
the instrument doing the measuring are both mine. If my shadow verifiers are more lenient than the real
ones, my attack looks great and scores zero. If they are stricter, I over-engineer. There is no fixed point
here: an evaluation is itself a model of the evaluator, and evaluating *that* just moves the regress one
level up. In the end I chose my attack on the strength of a rule I could read in the homework text (the
field enumeration) rather than on the strength of my own measurements — which is an admission that the
measurement was never really evidence.

## 6. The red team's knowledge is the fraudster's knowledge

I built the detector, and that is exactly what made the attack easy: I knew where I had not been told to
look. The homework makes this literal by scoring both halves, but the underlying paradox is general. The
skill of explaining why a system is trustworthy is the same skill as finding where it isn't, and it cannot
be handed to one person and withheld from another. What actually separates them is not the model's
knowledge but the system around it: whether a human ever sees the field list, whether the confidence number
is allowed to travel without its scope attached, whether anything outside the rubric is checked at all.

## What I would actually change

Three things, none of them about models. Publish the scope alongside the score, so that 0.9 reads as
"0.9 within these eleven fields" and a reviewer can see the two fields nobody compared. Treat the rubric as
an attack surface and audit it as one — every leniency clause is a specification for the next exploit.
And spend scepticism on the arithmetic, not on the text that is shouting. My own verifier got all of this
wrong while being, by its own measure, 90% confident.
