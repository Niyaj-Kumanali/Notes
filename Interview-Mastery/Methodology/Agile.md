# Agile

---

## Overview

- **Definition:** Agile is a software development methodology based on iterative development, cross-functional teams, and continuous feedback. Rooted in the Agile Manifesto (2001).
- **Why It Exists:** Traditional waterfall approaches failed to handle changing requirements, had long feedback cycles, and delivered software that no longer met user needs. Agile enables rapid adaptation, early delivery, and continuous customer involvement.
- **Key Concepts:** **Agile Manifesto** (4 values, 12 principles), **Iterations** (timeboxed development cycles), **Ceremonies** (standups, planning, review, retro), **User Stories**, **Velocity**, **Definition of Done**.

---

## Core Concepts

### Agile Manifesto — Four Values

1. **Individuals and interactions** over processes and tools
2. **Working software** over comprehensive documentation
3. **Customer collaboration** over contract negotiation
4. **Responding to change** over following a plan

### 12 Agile Principles

1. Satisfy customer through early/continuous delivery
2. Welcome changing requirements, even late in development
3. Deliver working software frequently (weeks, not months)
4. Business people and developers work together daily
5. Build projects around motivated individuals, trust them
6. Face-to-face conversation is most efficient
7. Working software is primary measure of progress
8. Sustainable development, constant pace
9. Continuous attention to technical excellence
10. Simplicity — maximizing work not done
11. Self-organizing teams produce best architectures
12. Regularly reflect and tune behavior

### Agile Methodologies

| Methodology | Key Practices | Best For |
|------------|---------------|----------|
| **Scrum** | Sprints, standups, retros | Complex product development |
| **Kanban** | Visualize flow, WIP limits, continuous delivery | Support, maintenance |
| **XP** | TDD, pair programming, CI | Engineering excellence |
| **Lean** | Value stream mapping, eliminate waste | Process optimization |
| **SAFe** | PI planning, ARTs, Lean Portfolio Mgmt | Enterprise scaling |

### Estimation Techniques

- **Planning Poker:** Fibonacci (1, 2, 3, 5, 8, 13, 21)
- **T-Shirt Sizing:** XS, S, M, L, XL
- **Affinity Mapping:** Group similar-sized items
- **Dot Voting:** Team votes on complexity
- **Bucket System:** Sort into predefined size buckets

### Key Metrics

| Metric | What It Measures | Target |
|--------|-----------------|--------|
| Velocity | Points per sprint | Consistent trend |
| Cycle Time | Start to finish | Decreasing |
| Lead Time | Request to delivery | Decreasing |
| WIP | Work in progress | < team size |
| Throughput | Items per sprint | Increasing |
| Burndown | Work remaining vs time | On track |

---

## Common Mistakes

- **Standups become status reports** — Focus on blockers instead of updates.
  - **Why it looks correct:** each person reporting what they did feels like accountability and gives managers the visibility they ask for. In practice, it turns a synchronization event into a one-way broadcast that hides blockers until they're critical — the person who needs help rarely raises their hand in a status roll call.
- **Retros without action** — Track action items, no improvement without follow-through.
  - **Why it looks correct:** having the conversation feels productive, and teams assume awareness alone drives change. Without tracked owners and deadlines, the same complaints reappear sprint after sprint, and cynicism replaces trust in the process.
- **Story points as performance metric** — Leads to gaming; use only for planning.
  - **Why it looks correct:** management wants data-driven evaluation, and points seem like an objective measure of developer output. In reality, points are a relative, team-specific estimate — comparing them across people incentivizes inflation, avoidance of complex work, and kills the collaboration that velocity actually depends on.
- **Too many WIP items** — Context switching kills productivity; enforce WIP limits.
  - **Why it looks correct:** starting more work feels like progress, and letting a developer sit idle while waiting on PR review seems wasteful. The hidden cost is task-switching overhead — a developer juggling 5 active items loses 30-50% of productive time to mental reload, which means every single item takes longer from start to finish.
- **Mid-sprint changes** — Loss of focus; protect the sprint goal.
  - **Why it looks correct:** the stakeholder's request feels genuinely urgent, and saying "no" to a VP or CEO seems like a career-limiting move. The production cost is a team that never finishes anything of substance — every mid-sprint change resets priorities, and the sprint goal becomes a decoration rather than a commitment, making delivery dates unpredictable for everyone.
- **No DoD** — Technical debt accumulates; create and enforce Definition of Done.
  - **Why it looks correct:** shipping faster without writing tests or documentation seems more productive in the moment, and "we'll come back to clean it up" always sounds reasonable. A missing DoD means every sprint ships untested, undocumented code — by sprint 5, the team spends roughly 50% of capacity fixing bugs from sprint 1 instead of building new features, and the "fast" approach becomes dramatically slower.
- **PO unavailable** — Wrong priorities result; PO must be accessible daily.
  - **Why it looks correct:** the PO seems busy with important stakeholders, and the team can "figure it out" and ask questions later. The production symptom is insidious rework — the team builds what they *think* is correct based on assumptions, which means the sprint review becomes a parade of rejected work, or worse, features ship that users don't actually need.
- **Team too large (over 9)** — Communication overhead; split the team.
  - **Why it looks correct:** adding more people should increase throughput — it is the obvious response to falling behind. The communication channels grow as n(n-1)/2: a 10-person team has 45 channels versus 15 for a 6-person team, so coordination overhead consumes the added capacity and velocity per person actually drops.
- **No automated testing** — Slow regression; invest in CI/CD.
  - **Why it looks correct:** manual testing "works" for small features, and writing automated tests takes time that could be spent on feature code. At 10-20 test cases, manual regression takes an hour; at 200+, it takes days — and the team eventually stops regression testing entirely, shipping every release with unknown breakage and praying nothing catastrophic was introduced.
- **Zombie Scrum** — Going through ceremonies without agile mindset; revisit values.
  - **Why it looks correct:** the team attends all ceremonies and uses the right vocabulary, so it *feels* like agile is working. The production symptom is zero improvement — velocity stagnates, the same problems recur every retro, and the team becomes cynical, treating agile as pointless overhead rather than a tool for continuous improvement.

---

## Key Design Considerations

- **Scaling frameworks:** SAFe (50+ teams), LeSS (3-8 teams), Nexus (3-9 teams), Spotify Model (tribes, squads, chapters, guilds)
- **When Agile fails:** No executive buy-in, teams not cross-functional, technical debt prevents rapid iteration, org structure conflicts with agility, too much process
- **Distributed teams:** Overlap working hours by 4+ hours, async communication, record ceremonies, rotate meeting times, quarterly face-to-face
- **DevSecOps in Agile:** Security stories prioritized, threat modeling in planning, security review in DoD, penetration testing per release
- **Waterfall vs Agile:** Evolutionary vs fixed requirements; incremental vs single release; continuous vs milestone customer involvement

---

## Real-World Scenarios

### Scenario 1: Startup Transitioning from Waterfall to Agile
A 50-person startup with 6-month release cycles realizes they ship features users don't want. Competitors ship weekly. **Transition:** Start with 2-week sprints, a PO from product, and an SM from engineering. The first 3 sprints are chaotic — estimates are off, the PO is overwhelmed, and standups are status reports. **Fix:** Reduce sprint to 1 week for faster feedback, add backlog refinement twice per week, train the PO on story writing, and enforce 15-min standup timeboxing. After 4 sprints, velocity stabilizes and predictability improves.

### Scenario 2: Distributed Team Across 4 Timezones
A team has members in San Francisco (UTC-8), London (UTC+1), Bangalore (UTC+5:30), and Sydney (UTC+11). Standups at 9 AM SF time are at 2:30 AM for Sydney. **Fix:** Establish 4-hour overlapping core hours (14:00-18:00 UTC). Rotate standup times weekly so each region shares the early/late pain half the time. Use async daily updates via Slack for non-overlap hours. Record sprint reviews and retros for those who can't attend live. Quarterly face-to-face for relationship building.

### Scenario 3: Velocity Drop After Microservices Migration
A team's velocity dropped 50% after migrating from a monolith to microservices. Sprints used to deliver 30 story points; now they deliver 12. **Diagnosis:** The team underestimated the learning curve for new tech (Docker, Kubernetes, event-driven patterns). DevOps overhead (CI/CD pipelines, service discovery, monitoring) was not accounted for. **Fix:** Dedicate one sprint to infrastructure (observability, deployment automation, developer experience). Reduce Definition of Done temporarily. Track velocity trend over 4 sprints — it should recover as the team gains proficiency.

---

## Scenario-Based Questions

1. **Q: You are the SM for a team where the PO keeps adding stories mid-sprint because "the CEO needs it by Friday." The team is demoralized. How do you handle this?**
   - **A:** Protect the sprint goal as non-negotiable. Have a private conversation with the PO explaining that mid-sprint changes undermine the team's autonomy and focus. Propose: the PO brings urgent items to the SM; if truly critical, the team swaps equal-sized stories (not adds). Escalate to management only after repeated violations. Track and report how many mid-sprint changes happen to build the case.

2. **Q: You join a team that's been doing agile for 2 years, but every retrospective comes up with the same action items and nothing changes. How do you break the cycle?**
   - **A:** Stop the retro on talk — move to action. Use the "Start/Stop/Continue" format with one binding action per person for the next sprint. Track actions in a visible board. Start each retro reviewing previous actions: "Did we do it? If not, why?" If organizational blockers prevent change, escalate with data (velocity impact, team satisfaction scores).

3. **Q: Management wants to measure developer productivity using story points per person. How do you respond?**
   - **A:** Story points are a team-relative estimate, not an individual productivity metric. Comparing points across individuals creates perverse incentives: (1) people inflate estimates, (2) people avoid complex work, (3) collaboration drops. Propose alternative metrics: cycle time, deployment frequency, team happiness, customer satisfaction. Point out that individual velocity doesn't exist in agile frameworks.

   - **Interview follow-up:** Management accepts your arguments against per-person velocity but responds: "Fine, we'll just measure team velocity and compare teams to each other." How do you respond? What makes velocity incomparable even between teams working on the same product?

4. **Q: Your team uses 2-week sprints but consistently finishes all work by day 8, then sits idle waiting for the sprint to end. What's happening and how do you fix it?**
   - **A:** The team is under-committing (sandbagging) or the PO isn't filling the backlog with enough refined items. Fix: (1) During sprint planning, use historical velocity as a guide, not a ceiling. (2) If work finishes early, pull the next item from the backlog (after PO confirms). (3) Consider shortening the sprint to 1 week to better match capacity. (4) Validate estimation — maybe the team has improved but estimates haven't adjusted.

5. **Q: A regulated fintech company wants to adopt agile but auditors demand requirements traceability, sign-offs, and documentation. How do you reconcile?**
   - **A:** Agile doesn't mean no documentation — it means the right documentation. Trace user stories → acceptance tests → test results. Use BDD (Gherkin scenarios) as living documentation. Automated CI/CD pipelines provide audit trails. Regulatory sign-offs become acceptance criteria in the Definition of Done. The key: documentation should be a byproduct of development, not a separate activity.

   - **Interview follow-up:** A production incident reveals that a critical compliance rule was never captured as an acceptance criterion in any story — it was "tribal knowledge" the senior dev always handled. The auditor flags this as a traceability gap. Who owns the fix — PO, SM, or the team? How would you redesign the process so tribal knowledge is systematically encoded without creating a documentation treadmill?

6. **Q: Your product owner is excellent at writing stories but terrible at prioritizing. The team builds perfect features that nobody uses. What do you recommend?**
   - **A:** Train the PO on value-based prioritization using Weighted Shortest Job First (WSJF) or Opportunity Scoring. Introduce outcome-based metrics (user adoption, task completion rate) instead of output-based (features shipped). Run user research sessions where the team observes real users. If the PO still can't prioritize, escalate — an incapable PO is a systemic risk.

7. **Q: A team of senior developers insists they don't need agile because "we already communicate well." They've been doing waterfall with 6-month releases. How do you convince them?**
   - **A:** Don't sell agile — sell outcomes. Ask: "How long does it take from idea to deployed software?" "When was the last time you pivoted based on user feedback?" "How much rework happens?" Run a 1-month pilot with a single product feature: 2-week sprint delivery vs their usual timeline. When they see user feedback after 2 weeks instead of 6 months, the value becomes self-evident.

8. **Q: Your 3 teams share one product backlog. Every sprint, the same high-priority items appear but nobody finishes them because each team picks partial work. How do you fix this?**
   - **A:** Split into team-specific backlogs organized by feature area or subsystem. Each team owns end-to-end delivery of items in their area. Use a shared Product Goal that aligns the teams. Have a weekly alignment meeting where teams negotiate dependencies. For items that cross teams, have one team own the item and the other team contributes as a dependency.

9. **Q: The organization has a "blameless culture" but retro action items never name specific people. The same problems recur. How do you make retros effective without violating psychological safety?**
   - **A:** Retros should focus on systems and processes, not individuals. Ask: "What in our process allowed this to happen?" instead of "Who made the mistake?" If naming is needed, use the "I" statement: "I feel we need clearer criteria for the Definition of Done." Assign action items to roles (SM, PO, Dev Team) rather than individuals. If issues persist, the SM should take system-level actions.

10. **Q: Your team adopted agile but the rest of the organization is still waterfall. The team delivers working software every 2 weeks, but it sits in QA for 4 weeks before release. What do you do?**
    - **A:** This is Water-Scrum-Fall. Fix: (1) Include QA in the sprint — shift testing left. (2) Automate regression tests so QA focuses on exploratory testing. (3) Create a release train — every sprint end triggers a deployment to a staging environment. (4) Negotiate with operations to allow continuous deployment or at least bi-weekly releases. (5) Make the case to management: the team delivers value in 2 weeks, but the organization delivers in 6 — the bottleneck is not the team.

    - **Interview follow-up:** Operations agrees to bi-weekly releases but the Change Advisory Board (CAB) requires 2 weeks of pre-approval for every production deployment. Your bi-weekly release still has a 2-week lead time before it. How do you work within this constraint without violating the CAB mandate, and what data would you gather to eventually challenge the 2-week pre-approval rule?

---

## Interview Questions

1. **What is the Agile Manifesto?**
   - **A:** Four values: individuals and interactions over processes and tools, working software over comprehensive documentation, customer collaboration over contract negotiation, responding to change over following a plan.

2. **What are the 3 key roles in Scrum?**
   - **A:** Product Owner (maximizes value), Scrum Master (coaches/coaches process), Development Team (self-organizing, builds the product).

3. **What is the difference between velocity and capacity?**
   - **A:** Velocity is historical — points delivered per sprint (past). Capacity is forecast — how much the team can do in the upcoming sprint considering leave, ceremonies, etc. (future).

4. **What is a user story? What is INVEST?**
   - **A:** A user story describes a feature from the user's perspective: "As a [user], I want [goal] so that [reason]." INVEST: Independent, Negotiable, Valuable, Estimable, Small, Testable.

5. **What is the purpose of a sprint retrospective?**
   - **A:** Inspect the team's process and adapt. The team discusses what went well, what could improve, and commits to concrete action items for the next sprint.

6. **What is technical debt and how does agile address it?**
   - **A:** Technical debt is the implied cost of future rework caused by taking shortcuts. Agile addresses it with continuous refactoring, Definition of Done, and allocating time in each sprint for quality improvements.

7. **What is the difference between Kanban and Scrum?**
   - **A:** Scrum uses fixed-length sprints with commitments. Kanban uses continuous flow with WIP limits. Scrum prescribes roles and ceremonies; Kanban is more flexible. Scrum is better for product development; Kanban for support/maintenance.

8. **What is a "Definition of Done" and why is it important?**
   - **A:** A checklist of criteria that must be met for a product increment to be considered done (e.g., code reviewed, tested, documented, deployed to staging). It ensures quality and transparency.

9. **What is the difference between a burndown and a burnup chart?**
   - **A:** Burndown shows remaining work vs time (does it trend to zero?). Burnup shows completed work vs total work (can show scope changes). Burndown is more common but burnup better communicates scope growth.

10. **What is the role of a Scrum Master?**
    - **A:** A servant leader who coaches the team on Scrum, removes impediments, facilitates ceremonies, protects the team from external disruptions, and helps the organization adopt agile values.

---

## Developer Recommendations

- **Keep estimates relative, not absolute** — Use story points (Fibonacci sequence) to compare effort, not hours. Absolute estimates are almost always wrong. Relative estimation accounts for uncertainty. Trade-off: points are meaningless outside the team; don't compare across teams.

- **Protect the sprint goal at all costs** — Every mid-sprint change dilutes focus and demoralizes the team. The PO and SM must be gatekeepers. Trade-off: sometimes you genuinely need to pivot (security emergency, customer outage). In those rare cases, cancel the sprint rather than corrupt it.

- **Enforce the Definition of Done strictly** — Skipping tests or code review to "go faster" creates technical debt that compounds. A relaxed DoD in sprint 1 leads to 50% rework in sprint 5. Trade-off: initially slower delivery velocity. Benefit: sustained velocity and quality over time.

  - **Production story:** A payments team skipped integration tests in their DoD for "just this sprint" to hit a regulatory deadline. The untested code introduced a rounding error in transaction fees. It went undetected for three months. By then, 47,000 customers had been overcharged ~$380,000. Remediation — refunds, regulatory fines, mandatory audit — consumed 14 developer-months and triggered an SEC inquiry. The "one sprint shortcut" took nine months to fully resolve.

- **Invest in automated testing and CI/CD** — Without automation, agile is just "fast waterfall." Automated regression tests enable the confidence to release every sprint. Trade-off: significant upfront investment in test infrastructure. Benefit: regression testing goes from 3 days to 3 minutes.

- **Use retros for genuine improvement, not ritual** — Action items without owners and follow-through create cynicism. Each retro should produce at least one concrete, tracked action. Trade-off: retros take 60-90 minutes from delivery. Benefit: continuous improvement compounds — 5% better each sprint is 2.8x better in a year.

  - **Production story:** A 40-person org ran retros every sprint for two years without tracking a single action item to completion. The same three complaints appeared in every retro — unclear requirements, unstable test environments, last-minute scope changes. The team stopped raising issues because "nothing ever changes." A developer satisfaction survey scored agile process at 1.8/5, and two senior engineers quit, citing "ceremonies without purpose." The retros had become a release valve without a repair mechanism — venting frustration without any pressure to fix the underlying causes.

- **Cross-functional teams are non-negotiable** — If every sprint requires a dependency on another team (DBAs, QA, DevOps), you're not agile. Build these capabilities into the team. Trade-off: harder to hire (T-shaped people). Benefit: no external dependencies block delivery.

- **Use metrics to improve, not to judge** — Velocity trends help with planning. Cycle time helps identify bottlenecks. Use them for team introspection, not management scorecards. Trade-off: metrics can be gamed. Benefit: data-driven process improvement.

- **Start agile adoption with 2-3 pilot teams** — Don't mandate org-wide agile transformation overnight. Pilot teams demonstrate value, develop coaches, and create patterns others can follow. Trade-off: slower rollout. Benefit: higher adoption rates and fewer "zombie Scrum" failures.
