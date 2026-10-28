# 9. Senior Behaviors: Leading, Estimating, Mentoring and Influencing

> Phase 5 · All 26 weeks (mostly practiced at work) · [← Phase index](README.md)

**Why this matters:** Technical skill gets you to Middle. What moves you to **Strong Middle and Senior** is how you multiply the team: you take ambiguous problems and turn them into delivered outcomes, you make other people better, and people trust your judgment. Nobody hands you these skills. You practice them deliberately, the same way you practiced async/await.

---

## 9.1 What changes as you grow

| Dimension | Middle | Strong Middle → Senior |
|---|---|---|
| **Scope** | features end to end | epics and cross-team problems; outcomes rather than tasks |
| **Ambiguity** | clarifies requirements when asked | **shapes** vague problems into plans, and asks "should we build this at all?" |
| **Technical decisions** | sound local decisions | system-level trade-offs, documented (design docs, ADRs) |
| **Quality** | good code and tests | sets standards; improves the *system* that produces quality (CI, reviews, guidelines) |
| **People** | helps when asked | mentors deliberately; reviews to teach; unblocks others first |
| **Communication** | clear status in stand-ups | adapts the message to the audience (engineers, product, management); writes well |
| **Operations** | fixes incidents with help | leads incidents and drives the follow-ups |

---

## 9.2 Leading a feature with 2+ developers

**1. Shape it (before any code)**

- Clarify the **goal and success metric** ("reduce abandoned checkouts by 10%"), not just the feature list.
- Write a short **design doc** (Phase 3, guide 9) and get it reviewed.
- Identify **risks and unknowns**, and timebox **spikes** for the scariest ones first.

**2. Break it down**

- Build a **walking skeleton** first: the thinnest end-to-end slice deployed to staging (UI → API → database), to flush out integration problems early.
- Split the rest into **vertical slices** that each deliver visible value, and that people can work on in parallel with few conflicts.
- Make dependencies explicit (an API contract agreed before both sides build).

**3. Run it**

- A visible board, a short daily sync on **blockers**, and a weekly demo.
- **Protect the team's flow:** answer their questions fast, review their PRs first, and take on the boring glue work yourself.
- Track risks openly, and escalate early. *Bad news early is a gift; bad news late is a crisis.*

**4. Land it**

- Use a rollout plan with a flag, monitoring and a rollback plan (Phase 4).
- Measure the success metric after release.
- Run a short retrospective, and share what you learned.

---

## 9.3 Estimation

**Estimates are ranges with a confidence level, not promises.**

| Technique | How |
|---|---|
| **Ranges** | "3–6 days, most likely 4". Say what would push it to the high end |
| **Three-point (PERT)** | `(optimistic + 4 × likely + pessimistic) / 6`, per task, then summed |
| **Break it down** | tasks over ~2 days hide unknowns, so split them further |
| **Spikes** | timebox research ("2 days to evaluate the payment SDK") before estimating the unknown part |
| **Reference class** | "the last 3 similar integrations took 8–12 days" beats optimism |
| **Re-estimate** | as unknowns resolve, update early and visibly |

> **The cone of uncertainty:** early estimates can be off by 2–4×. Communicate that honestly, and narrow it by retiring risks first.

---

## 9.4 Communication

### Status update template (weekly, async)

```markdown
**Checkout v2: week 3 of ~5** · Status: AMBER (at risk)
- Done: walking skeleton in staging; payment provider sandbox integrated
- Next: seat-hold expiry, failure UX
- Risk: provider webhooks arrive out of order → +2–3 days for idempotent handling (design doc updated)
- Ask: decision from Product on "pay later" fallback by Thursday
```

### Writing for decision-makers: BLUF (bottom line up front)

> *"Recommendation: delay the launch by one week. Load tests show checkout fails above 300 RPS, and the sale expects 800. Fix: connection pooling plus caching (3 days); the details are below."*

- Lead with the **decision or recommendation**, then the evidence, then the details.
- Translate technical issues into **impact**: users, money, risk, time.
- Offer **options with trade-offs** instead of a single "no".

---

## 9.5 Mentoring

| Do | Avoid |
|---|---|
| Ask guiding questions ("what happens if this message arrives twice?") | taking the keyboard and fixing it for them |
| Pair on hard tasks; let them drive | assigning only trivial tasks |
| Use code review to **teach**: explain why, and link to docs | nitpicking style that tools should enforce |
| Agree on growth goals and check in every couple of weeks | ad-hoc help only when they're stuck |
| Share your own mistakes and how you learned | pretending you always knew |
| Write onboarding docs and runbooks | knowledge that lives only in your head |

**A good sign you're mentoring well:** they come with *options* instead of just questions, and they start reviewing others' code with the same lens you taught them.

---

## 9.6 Driving technical improvements

Most improvements fail because of *how* they're proposed, not because of the idea.

1. **Find the pain with data:** "CI takes 25 minutes; developers wait ~3 h/week each; flaky tests cause 1 in 6 re-runs."
2. **Propose small:** a one-page RFC with the problem, the options, a recommendation and a reversible first step.
3. **Pilot** it on one repository or one team, and **measure** the result.
4. **Roll out** with documentation and help, and report the outcome ("CI 25 → 9 min, flaky re-runs down 80%").
5. **Celebrate the team**, not yourself. That's how you get support for the next improvement.

### Technical debt

- Keep a **debt register**: the item, its impact (incidents, slowness, risk), the effort and the owner.
- Tie debt to business outcomes ("this module causes 40% of checkout bugs").
- Negotiate a steady capacity share (for example, 15–20%) instead of a heroic "refactoring sprint".

---

## 9.7 Influence without authority

- **Build trust first:** deliver reliably, admit mistakes, and give credit generously.
- **Understand others' goals.** Product wants outcomes, operations wants stability, and security wants risk reduced. Frame proposals in *their* terms.
- **Write it down.** A clear RFC or design doc scales your influence beyond the meetings you attend.
- **Disagree and commit.** Argue your case with evidence, then fully support the decision once it's made.
- **Find allies** before big meetings: pre-wire proposals with key people one-on-one.

---

## 9.8 Operational ownership

- Join the **on-call rotation**, and treat incidents as learning (Phase 4, guide 9).
- Lead at least one **blameless postmortem** and drive its action items to completion.
- Make your services **boring to operate**: good alerts, runbooks, dashboards, and tested rollbacks.

---

## 9.9 Visibility and your career

| Practice | Why |
|---|---|
| **Brag document** (started in Phase 0), updated weekly | the raw material for reviews and promotion; memory fades fast |
| **Impact statements** in STAR form with numbers | "Led checkout v2 (3 devs); p95 −60%, abandonment −12%, zero incidents during the sale" |
| **Talks and posts** (2–4 in this phase) | build reputation; they force deep understanding |
| **Ask for feedback** regularly: "What would make you see me as ready for the next level?" | removes guesswork |
| A **promotion conversation** with evidence, not hopes | your lead needs material to argue for you |

### Talk or blog-post outline

```markdown
1. Hook: the problem in one sentence, with a number ("Our checkout fell over at 300 RPS")
2. Context: just enough for the audience to follow
3. What we tried, including what didn't work
4. What worked, with code, diagrams and before/after numbers
5. Lessons and when NOT to do this
6. Links: repo, docs, ADRs
```

---

## Behavioral interview questions (Senior-leaning)

Prepare STAR stories (Situation, Task, Action, Result) for each of these:

1. Tell me about a project you led. How did you break it down, and how did you handle a slip?
2. Describe a technical decision you pushed for that others disagreed with.
3. Tell me about a time you mentored someone. What changed for them?
4. Describe the worst production incident you were part of. What did you change afterward?
5. Tell me about a time you said "no" to a stakeholder, or proposed a cheaper alternative.
6. Describe an improvement you drove without being asked. How did you get buy-in?
7. Tell me about a time you were wrong. How did you find out, and what did you do?

---

## Exercises (at work, over the 26 weeks)

1. Lead one feature involving 2+ developers using the shape → break down → run → land steps.
2. Send a weekly status update in the template format for that feature.
3. Mentor one junior developer for three months, with agreed goals and fortnightly check-ins.
4. Drive one measurable improvement (CI time, flaky tests, alert noise, an onboarding doc).
5. Give one internal talk and publish one external blog post ([Lab 10](11-labs.md#lab-10-share-what-you-learned)).
6. Update your brag document weekly, and have the promotion conversation at the end of the phase.

## Self-check

- [ ] I can shape, break down, run and land a multi-person feature.
- [ ] I estimate with ranges and communicate risk early.
- [ ] I write status updates and recommendations that decision-makers can act on.
- [ ] I mentor deliberately and drive improvements with data.
- [ ] I have impact statements with numbers ready for reviews and interviews.

## Further reading

- *The Staff Engineer's Path* (Tanya Reilly): leading without being a manager.
- *An Elegant Puzzle* (Will Larson), and his blog *Irrational Exuberance*.
- *Software Estimation: Demystifying the Black Art* (Steve McConnell).
- *The Manager's Path* (Camille Fournier), the tech-lead chapters.
