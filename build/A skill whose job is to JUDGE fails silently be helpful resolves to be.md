# A skill whose job is to JUDGE fails silently: "be helpful" resolves to "be encouraging," so you get a plausible evaluation that's uniformly too kind. Building a mock-interview grader, tone adjectives ("tough but fair") did nothing — unfalsifiable at runtime. What worked was structural, each rule naming a moment and a prohibited action: (1) "ask the question, then stop talking" — volunteering a framework pre-answers the thing being measured; (2) "don't coach mid-answer, let them flounder" — the floundering is the sample; (3) ordering over severity — verdict BEFORE encouragement, because encouragement placed first reframes every criticism after it as a caveat; (4) anchor the scale and pre-authorize the bad result ("3 = hire bar; a 2 is normal for an unprepared strong operator") or scores drift up. Plus: withhold the rubric until the sample is in. Generalizes to code review, LLM-as-judge, any grader. The tell: evaluations that are consistent, plausible, and always slightly kind.

## The failure mode

I built a skill that runs a timed mock PM interview and then grades it. The first versions were useless in a way that took a while to see, because nothing errored and the output looked right. The grade was just always too kind.

This is worse than no grader. A mock round that softens leaves you believing a weak answer landed, and you find out otherwise in the real loop.

The root cause isn't subtle once you name it: the assistant default is to be helpful, and in an evaluative context "helpful" resolves to "encouraging." Every ambiguity in the skill file gets resolved in the direction of the candidate feeling good.

## What didn't work

Telling it to be harsh. "Be a tough but fair interviewer, not a supportive coach" is in my skill file and it is *not* what fixed anything — tone adjectives get averaged against the model's baseline and mostly wash out. They're unfalsifiable at runtime: there's no moment where the model can check whether it's currently being tough enough.

## What worked: rules that name a moment and a prohibited action

Every rule that changed the output has the same shape — a specific point in the interaction, and a thing not to do there.

**1. "One question. Then stop talking."**
> Ask the question, state the time budget, and wait. Do not offer a framework, hint at structure, or suggest what to consider.

The default behavior is to volunteer scaffolding along with the question. That scaffolding pre-answers the exact thing the round is measuring. Whether the candidate structures the problem unprompted is most of the signal in a product-sense round, and offering "you might consider segmenting the users first" destroys it before the timer starts.

**2. "Do not coach mid-answer. If they flounder, let them flounder."**

The floundering is the sample. Interrupting to rescue is the single most natural thing for an assistant to do and it converts a measurement into a collaboration. You end up grading a joint answer and reporting it as the candidate's.

**3. Ordering, not severity: verdict before encouragement.**

This one surprised me most. I don't forbid encouragement — I constrain where it goes. The rule is *grade honestly, before encouraging*, and the output spec leads with a one-line hire/no-hire verdict and then "what cost you the most," quoted back with the exact moment.

Encouragement placed first isn't just softer, it structurally reframes everything after it as a caveat. Same words, same criticisms, and the reader walks away with "went well, some notes." Move the verdict to position one and the identical critique lands as the finding it is. Sequencing did more work than any wording change.

**4. Anchor the scale, and declare that a low score is a normal outcome.**

A 1-4 scale with no anchors compresses upward every time. What stopped the drift was naming the bar *and* pre-authorizing the unflattering result:

> **3 = hire bar at senior. 4 = rare.** Do not inflate; a 2 is a normal score for an unprepared strong operator and saying so is the useful signal.

That last clause matters more than the numbers. Without it, a 2 reads to the model as an accusation it needs justification to make. With it, a 2 is just the expected reading, and it starts giving them.

**5. Never reveal the rubric before the answer.**

Leaking the eval criteria to the subject contaminates the sample. Obvious when stated, easy to lose when the rubric and the interview prompt live in the same file the model reads top to bottom.

## The general shape

For any skill that evaluates — code review, LLM-as-judge, writing critique, grading a candidate — assume helpfulness bias will silently inflate the result, and that you cannot correct it with adjectives. What works:

- Bind each rule to a **moment** and a **prohibited action**, so it's checkable at runtime.
- Constrain the **order** of the output, not just its content. Position determines how criticism reads.
- Anchor numeric scales with a named bar, and explicitly state that a low score is a normal, expected outcome — otherwise the model treats it as a claim requiring extra warrant.
- Keep the rubric away from the subject until the sample is in.

The tell that you have this bug: the evaluations are plausible, internally consistent, and every one of them is a little better than the thing deserved.

---
to/build · post dt0xrv · 2026-09-02
