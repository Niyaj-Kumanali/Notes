# Scrum

---

## Overview

- **Definition:** Scrum is a lightweight agile framework for developing, delivering, and sustaining complex products. Based on empiricism (transparency, inspection, adaptation), it uses fixed-length iterations called sprints.
- **Why It Exists:** Traditional project management fails for complex product development. Scrum provides a structured yet flexible framework that embraces uncertainty through iterative delivery, frequent inspection, and team self-organization.
- **Key Concepts:** **Three Pillars** (Transparency, Inspection, Adaptation), **Three Roles** (PO, SM, Dev Team), **Five Events** (Sprint, Planning, Daily, Review, Retro), **Three Artifacts** (Product Backlog, Sprint Backlog, Increment), **Three Commitments** (Product Goal, Sprint Goal, Definition of Done).

---

## Core Concepts

### The Three Pillars

- **Transparency:** Process aspects visible to those responsible
- **Inspection:** Artifacts inspected frequently for variances
- **Adaptation:** Process adjusted quickly when deviations detected

### Scrum Values

| Value | Description |
|-------|-------------|
| Commitment | Team commits to achieving sprint goals |
| Courage | Team has courage to do the right thing |
| Focus | Team focuses on sprint work |
| Openness | Team is open about work and challenges |
| Respect | Team members respect each other |

### Team Structure

```
Product Owner (1) — maximizes value, manages backlog
Scrum Master (1) — coaches team, removes impediments
Dev Team (3-9) — self-organizing, cross-functional
```

### Five Events

| Event | Timebox | Purpose |
|-------|---------|---------|
| **Sprint** | ≤ 1 month | Container for all other events |
| **Sprint Planning** | max 4 hrs / 2wk | What + How for the sprint |
| **Daily Scrum** | 15 min | Sync + plan next 24h |
| **Sprint Review** | max 4 hrs / 2wk | Demo completed work, get feedback |
| **Sprint Retrospective** | max 3 hrs / 2wk | Inspect + adapt team process |

### Artifacts & Commitments

| Artifact | Commitment | Description |
|----------|------------|-------------|
| Product Backlog | Product Goal | Ordered list of everything needed |
| Sprint Backlog | Sprint Goal | Selected items + delivery plan |
| Increment | Definition of Done | Usable product at sprint end |

### Sprint Length Trade-offs

| Duration | Pros | Cons | Best For |
|----------|------|------|----------|
| 1 week | Fast feedback, easy planning | High overhead | Early stage, uncertain |
| 2 weeks | Balance of predictability | Moderate overhead | Most teams |
| 3-4 weeks | Longer to deliver | Slow feedback | Established products |

### Key Metrics

- **Velocity:** Points delivered per sprint (trend matters, not value)
- **Sprint Goal Success Rate:** % of sprints achieving goal (> 80% target)
- **Predictability:** Planned vs actual variance (< 15% target)
- **Happiness Index:** Team morale 1-5 (> 4.0 target)
- **Escaped Defects:** Bugs found in production (decreasing trend)

---

## Common Mistakes

- **No PO availability** — Wrong priorities; PO accessible daily required.
  - **Why it looks correct:** the PO seems trustful of the team's judgment, and giving the team autonomy feels like empowerment. The production symptom is invisible misalignment — the team builds what they *assume* is correct, and the sprint review becomes a parade of rejected work or, worse, features ship that miss the actual market need.
- **SM also develops** — Conflict of interest; dedicated SM needed.
  - **Why it looks correct:** the SM is often a senior engineer who knows the codebase best, and "helping out" seems like being a team player. The conflict of interest is structural — the SM cannot impartially facilitate ceremonies, remove impediments, or hold the process accountable when they are also accountable for delivering story points.
- **Changing sprint scope** — Loss of focus; protect sprint goal.
  - **Why it looks correct:** the new work feels genuinely urgent, and accommodating it seems responsive to the business. The production cost is that the sprint goal becomes a decoration, the team's autonomy erodes, and velocity data becomes unreliable for planning — a sprint with 40 points of planned work plus 20 points of mid-sprint adds tells you nothing about true capacity.
- **No sprint goal** — Aimless sprint; define goal each sprint.
  - **Why it looks correct:** the team just works through the backlog top-to-bottom, which seems orderly and productive. Without a sprint goal, the team has no basis for trade-off decisions when unexpected work arises — every item feels equally important, so nothing gets deprioritized, and the team delivers a collection of unrelated tasks rather than a coherent increment.
- **Missing retros** — No improvement; make mandatory.
  - **Why it looks correct:** skipping a retro frees up 90 minutes for "real work," which looks like a productivity win. The cumulative cost is that the team never pauses to improve — the same inefficiencies compound sprint after sprint, and the team gets faster at doing the wrong things without ever correcting course.
- **Standup > 15 min** — Waste; focus on blockers.
  - **Why it looks correct:** complex work requires detailed discussion, and cutting off conversation seems anti-collaborative. The math is brutal: a 30-minute standup with 6 people costs 3 person-hours per day — that is 15 hours of team time per week for what should be a 15-minute synchronization check, not a problem-solving session.
- **Team too large/small** — Inefficiency; 3-9 members.
  - **Why it looks correct:** adding people when the team is behind seems like the obvious fix, while keeping a small team seems like avoiding overhead. The trade-off is invisible until you measure: a 2-person team lacks cross-functionality (blocked whenever one person is out), while an 11-person team has 55 communication channels versus 15 for a 6-person team — coordination overhead consumes any throughput gain.
- **No DoD** — Technical debt; create and enforce DoD.
  - **Why it looks correct:** it speeds up individual story completion when there are no quality gates to pass. The long-term cost is that "done" means different things to different people — the developer considers it done when code compiles, the tester when tests pass, and the PO when acceptance criteria are met — causing friction and rework at every handoff boundary.
- **No estimation history** — Inaccurate; track velocity.
  - **Why it looks correct:** estimation feels unreliable anyway — "what's the point of tracking something we always get wrong?" — so skipping it seems pragmatic. Without historical data, every sprint planning session starts from optimism rather than evidence, and the team chronically over-commits, demoralizing themselves sprint after sprint when they miss their own targets.
- **No backlog refinement** — Poor planning; 10% sprint time.
  - **Why it looks correct:** refinement time could be spent coding, and the PO can "just add items directly to the sprint." The result is that sprint planning becomes a discovery session instead of a commitment session — the team discovers requirements, technical unknowns, and missing acceptance criteria during planning, turning a 4-hour event into an exhausting and inaccurate negotiation.
- **Zombie Scrum** — Going through motions; revisit Scrum values.
  - **Why it looks correct:** the team attends all ceremonies and uses the right vocabulary — standups happen, retros happen, boards are updated. The symptom is invisible to observers: velocity is flat, the same problems recur every retro, team satisfaction is low, and nobody can articulate *why* they do any of the ceremonies beyond "that's what Scrum says."
- **Water-Scrum-Fall** — Agile ceremonies, waterfall mindset; true cross-functional teams.
  - **Why it looks correct:** the team adopted agile ceremonies while the rest of the organization operates in sequential phases — it looks like progress. The production cost is that the team delivers every 2 weeks but the organization releases every 3 months, so the business sees no acceleration and management concludes "agile doesn't work here," threatening the entire transformation.
- **Hero culture** — Same person saves every sprint; spread work, coach others.
  - **Why it looks correct:** the hero delivers results and gets public recognition, which seems like a positive motivator and an example for others. The hidden cost is that the rest of the team disengages ("why bother, Alex will fix it"), the hero burns out or leaves, and the bus factor stays at 1 — if the hero is out sick, the entire sprint collapses.
- **Proxy PO** — Delegate makes decisions; PO must be available.
  - **Why it looks correct:** the real PO is senior and busy, and sending a delegate seems like reasonable delegation of authority. The delegate lacks decision power, so every question becomes "let me ask my manager" — the team stalls on every prioritization decision, and the sprint backlog becomes a wish list rather than a committed plan with clear trade-offs.

---

## Key Design Considerations

- **Scrum Master vs Project Manager:** SM is servant leader focused on process; PM manages budget, timeline. One person cannot do both.
- **When not to use Scrum:** Maintenance-only teams (use Kanban), highly uncertain research (use XP/Lean Startup), very small teams (1-2 people)
- **Distributed Scrum:** Core hours overlap, async updates, recorded ceremonies, rotate meeting times
- **Scaling:** Scrum-of-Scrums for multi-team coordination; Nexus (3-9 teams on same product), LeSS (3-8 teams), SAFe (enterprise)
- **Sprint Health Assessment:** Green (consistent velocity, goal achieved, no defects, happiness ≥ 4), Yellow (velocity drop > 20%, partial goal, minor defects), Red (velocity drop > 40%, goal missed, major defects)

---

## Real-World Scenarios

### Scenario 1: ScrumBut — Skipping Retros, No PO Availability
- A team claims to use Scrum but the PO is never available for backlog refinement, the SM is also a developer, and retros were cancelled 6 months ago. Sprint goals are rarely met, and the team feels like they're in a "feature factory." **Diagnosis:** This is Zombie Scrum — going through ceremonies without the mindset. **Fix:** Dedicated SM (no development work), PO must attend planning/refinement/review weekly, reinstate retros with a strict action-item tracking system. Coach the team on the "why" behind each Scrum event.

### Scenario 2: Twenty-Person Scrum Team
- Twenty developers are in a single Scrum team. Standups take 45 minutes, sprint planning takes 8 hours, and communication overhead is crippling. **Fix:** Split into three feature-aligned Scrum teams (6-7 members each). Each team has its own backlog and sprint goal but shares a common Product Goal. Use a Scrum-of-Scrums for cross-team coordination (3 representatives, 3 times a week, 15 minutes). One PO works with three APIOs (Associate POs) per team.

### Scenario 3: Sprint Reviews Nobody Attends
- The team holds sprint reviews on Friday at 4 PM. Stakeholders rarely attend. When they do, they give vague feedback ("looks good"). The team feels demotivated. **Fix:** Move the review to Tuesday at 10 AM. Invite stakeholders individually (not a blanket calendar invite). Prepare a structured demo: show one working feature, share metrics (velocity, quality, customer feedback), then ask specific questions ("Would this feature solve your problem? What's missing?"). Record sessions for absent stakeholders.

## Use Cases

- **Product development with clear priorities** — building a SaaS product where the Product Owner prioritizes features
  - Fixed-length sprints (2 weeks) deliver a potentially shippable increment. PO prioritizes backlog. Team self-organizes to meet the sprint goal.
  - **Avoid when:** work is unpredictable or reactive (production support, incident response) — Kanban's continuous flow is better suited.

- **Team building and trust** — new teams learning to work together
  - Scrum ceremonies (planning, daily standup, review, retro) build communication habits. Retrospectives provide structured improvement. Sprint commitment creates team accountability.
  - **Avoid when:** the team is fully remote and across 12+ timezones — synchronous ceremonies are harder; consider Scrumban with fewer prescribed meetings.

- **Predictability and transparency** — stakeholders want to know when features will be delivered
  - Velocity tracking over multiple sprints enables release forecasting. Sprint reviews demonstrate progress. Burndown charts show real-time sprint health.
  - **Avoid when:** the team is constantly interrupted by urgent requests — Scrum assumes the team has focus; protection from interruptions is the SM's job.

- **Multi-team coordination** — large products requiring multiple Scrum teams
  - Scrum-of-Scrums coordinates cross-team dependencies. Shared Product Goal aligns teams. Feature teams own end-to-end delivery. Release trains synchronize cadences.
  - **Avoid when:** teams have no interdependencies — independent Scrum teams without coordination overhead are more efficient.

- **Sprint goal focus** — shielding the team from mid-sprint changes and scope creep
  - Commitment to the sprint goal prevents distractions. PO may swap items only if the sprint goal is not affected. The team owns how to achieve the goal.
  - **Avoid when:** stakeholder trust is low — they may resist giving up mid-sprint change authority until they see consistent delivery.

---

## Scenario-Based Questions

- **Q: You are a Scrum Master for a team where the PO treats the team as "resources" and assigns individual tasks. The team is demotivated and self-organization is dead. How do you restore Scrum?**
   - **A:** Coach the PO on the difference between "commanded" and "self-organizing" teams. Explain that the Dev Team commits to the Sprint Goal, not individual tasks. Introduce swarm-based commitment: the team collectively owns all items. If coaching fails, facilitate a retro where the team shares how the assignment model affects them. Escalate to management with data: task-assigned teams have 30% lower velocity and 50% higher turnover.

   - **Interview follow-up:** The PO pushes back: "I'm accountable for delivery. If I don't assign tasks, how do I know the right people are working on the right things?" How do you convince them that self-organization does not mean chaos, and what mechanism replaces individual task assignment?

- **Q: Your team's sprint reviews are poorly attended (2 of 10 stakeholders show up). Those who attend give vague feedback. The team feels they're demoing to an empty room. How do you fix this?**
   - **A:** Change the format and timing. (1) Move reviews to Tuesday/Wednesday at 10 AM — avoid Monday/Friday. (2) Send personalized invitations with a 1-line teaser of what will be demoed. (3) Shorten the demo to 15 minutes — show one working feature end-to-end. (4) Ask specific questions: "Does this solve your problem? What's missing?" (5) Record and share a 5-min video for absent stakeholders, asking for async feedback by Thursday.

- **Q: The team's velocity has been flat for 8 sprints despite the team growing from 5 to 8 members. What's happening and how do you diagnose?**
   - **A:** Brooks' Law — adding people to a late project makes it later. The new members need onboarding, create communication overhead, and the existing team spends time ramping them up. Diagnose: (1) Check sprint-by-sprint story completion. (2) Survey the team on productivity blockers. (3) Look at cycle time — it may have increased. Fix: improve onboarding, pair new members, ensure clear interfaces between work areas. Consider splitting into two teams.

- **Q: Management wants 2-week sprints, but the team works on a safety-critical medical device where every change requires regulatory review. Sprints end but releases take 3 months. How do you adapt Scrum?**
   - **A:** Use Scrum for development (2-week sprints for building features) but accept that releases follow a separate regulatory cadence. Create a "release train" — accumulated increments are submitted for regulatory review together. The Definition of Done includes all regulatory documentation and verification. Track "lead time" (idea to patient) separately from sprint delivery. This is not Water-Scrum-Fall — it's Scrum in a regulated context.

   - **Interview follow-up:** A critical patient-safety bug is discovered in an increment that already passed regulatory review and is awaiting release. Fixing it triggers a new regulatory review cycle, adding 3 months to the fix reaching patients. How would you design the release train to handle emergency fixes without breaking the regulatory process?

- **Q: A senior stakeholder often attends the Daily Scrum and asks detailed technical questions, turning it into a 30-minute status meeting. The team is afraid to ask her to leave. What do you do as SM?**
   - **A:** First, talk to the stakeholder privately: "The Daily Scrum is for the team to synchronize, not for status updates. Your attendance makes the team feel they're reporting to you." Offer an alternative: a 15-min weekly sync where you brief her. If she insists on attending, establish a ground rule: stakeholders listen only, speak only after the 15-min timebox. Put a timer visibly on the table. If necessary, physically move the standup to a location the stakeholder can't easily access.

- **Q: The team uses Scrum but the Sprint Backlog is never updated after planning. Stories stay "To Do" the whole sprint and on the last day they all move to "Done." What's missing?**
   - **A:** The team isn't using the Sprint Backlog as a living plan. Fix: (1) Visualize work with a physical or digital board updated daily. (2) Break stories into tasks (hours or smaller units). (3) In the Daily Scrum, reference the board — "I'm working on task X in story Y." (4) Use a burn-down chart visible to the team. (5) The SM should ask "what changed on the board today?" not "what did you do?"

- **Q: Two Scrum teams on the same product keep stepping on each other — modifying the same files, causing merge conflicts, and breaking each other's features. How do you coordinate?**
   - **A:** The teams lack architectural boundaries. Fix: (1) Define clear module ownership — each team owns specific components. (2) Establish API contracts between modules. (3) Implement CI that runs both teams' tests. (4) Have a weekly cross-team alignment meeting (Scrum-of-Scrums). (5) If conflict persists, merge into one team or restructure teams by feature vertical (not technical layer).

- **Q: A team member consistently delivers low-quality work — no tests, no documentation, frequent production bugs. The team is frustrated but avoids confrontation. How does the SM handle this?**
   - **A:** Privately coach the individual first: "I noticed some issues with the last few stories. Let's pair on the next one to see how we can improve." If no improvement, make quality a team conversation in retro, not personal criticism: "Our escaped defect rate went up. What can we change in our process?" Enforce the Definition of Done strictly — the team should not accept stories that don't meet DoD. If all fails, escalate to management with specific evidence (failed builds, production incidents).

- **Q: The product has 10 microservices. The team has 5 developers, and each sprint they must touch all 10 services to ship a feature. Sprints are chaotic. How do you structure the work?**
   - **A:** The team is a "feature team" touching too many surfaces. Fix: (1) Reduce scope — each sprint, focus changes to at most 3 services. (2) Create release trains for multi-service features across sprints. (3) If the product truly requires touching all 10 services per feature, the architecture is wrong — consolidate services. (4) Consider monorepo with shared CI to reduce cross-service overhead. (5) The PO should break features into smaller MVPs that affect fewer services.

- **Q: The team consistently over-commits and under-delivers. They're optimistic but demoralized when they fail the sprint goal every sprint. How do you improve forecasting?**
   - **A:** This is the "planning fallacy" — humans underestimate effort. Fix: (1) Use historical velocity as a hard ceiling for commitment, not a target. (2) Apply "reference class forecasting" — compare new work to similar past work. (3) Add a 30% buffer for unknowns. (4) Break large stories (< 8 points) into smaller ones. (5) After the sprint, do a "commitment vs delivery" analysis in retro. Celebrate when the team commits less but delivers fully — it builds confidence, not cynicism.

   - **Interview follow-up:** The team tries reference class forecasting, but every story is "unique" — they have no past work that resembles the current feature. Without historical analogues, how do you still produce a reliable forecast, and what data would you start collecting today to make future forecasts more accurate?

---

## Interview Questions

- **What are the three pillars of Scrum?**
   - **A:** Transparency (process visible), Inspection (artifacts inspected often), Adaptation (process adjusted when needed).

- **What are the five Scrum events?**
   - **A:** Sprint, Sprint Planning, Daily Scrum, Sprint Review, Sprint Retrospective. The Sprint is a container for all others.

- **What is the product backlog and who owns it?**
   - **A:** An ordered list of everything needed for the product. Owned by the Product Owner, who prioritizes and refines it continuously.

- **What is the Sprint Goal?**
   - **A:** A single objective for the sprint that unifies the selected backlog items. It answers "why are we doing this sprint?" and guides decision-making when priorities shift.

- **What is the difference between the Sprint Review and the Sprint Retrospective?**
   - **A:** Sprint Review inspects the product (what was built) with stakeholders. Sprint Retrospective inspects the process (how the team works) — team only.

- **What is a product increment?**
   - **A:** A usable, potentially releasable product at the end of each sprint. Each increment is additive to all previous increments.

- **What happens if a developer is blocked during a sprint?**
   - **A:** They raise it in the Daily Scrum. The SM removes impediments. If the impediment can't be resolved quickly, the team swarms to help or the blocked story is swapped out.

- **Who estimates work in Scrum?**
   - **A:** The Development Team — they will do the work, so they estimate. The PO provides context; the SM facilitates.

- **What is the Definition of Done?**
   - **A:** A checklist of criteria that must be met for work to be considered complete. It ensures quality and is agreed upon by the team, not imposed externally.

- **What is the recommended size of a Scrum team?**
   - **A:** 3-9 development team members. Smaller teams may lack cross-functionality; larger teams suffer from communication overhead and reduced collaboration.

---

## Developer Recommendations

- **The Sprint Goal is the team's north star** — Without a clear Sprint Goal, the team just works through a task list. The goal enables the team to make decisions autonomously when unexpected work arises. Trade-off: defining a meaningful goal takes time during planning. Benefit: the team has direction, motivation, and a clear "done" criterion.

- **Invest in backlog refinement as much as sprint execution** — Poorly refined backlog items lead to planning surprises and incomplete sprints. Dedicate 10% of sprint capacity to refinement. Trade-off: less time for delivery. Benefit: sprint planning becomes predictable (no surprises), and the team can estimate accurately.

- **The Daily Scrum is for the team, not management** — Anyone can attend but only the team speaks. It synchronizes, identifies blockers, and re-plans the next 24 hours. Trade-off: 15 min/day of synchronized time. Benefit: prevents the "I didn't know you were blocked" problem that wastes days.

- **Retros are the engine of continuous improvement** — Without action-oriented retros, the team repeats the same mistakes. Each retro must produce at least one SMART action item with an owner. Trade-off: 60-90 min per sprint. Benefit: compounding improvement — 1% better each sprint is 67% improvement in a year.

  - **Production story:** A team of 9 ran retros every sprint but never assigned owners or deadlines to action items. The same top issue — "unstable staging environment" — appeared in 12 consecutive retros. Developers spent an estimated 6 hours per sprint hunting environment-related failures instead of building features. Over two quarters, that was ~575 hours burned on a problem everyone agreed existed but nobody was accountable to fix. When the environment finally crashed during a customer demo, the VP of Engineering asked "Why wasn't this fixed?" The answer: it was everyone's problem, which means it was no one's problem.

- **The PO must be accessible daily** — A part-time or unavailable PO creates waste: the team builds the wrong things, blocks on decisions, or proceeds without clarity. Trade-off: the PO's full attention is expensive. Benefit: the team never waits for decisions and builds the right features.

- **Enforce the Definition of DoD, not sprint scope** — Quality is non-negotiable. If the team can't finish all items within quality standards, deliver fewer items at high quality rather than more items with technical debt. Trade-off: lower velocity initially. Benefit: sustained velocity without quality-related slowdowns.

  - **Production story:** An e-commerce team relaxed their DoD to exclude load testing for a Black Friday feature, reasoning they could "load test after the sprint." The feature shipped without load testing, crashed under 40% of projected Black Friday traffic at 2 AM on the busiest shopping day of the year. The site was down for 47 minutes — estimated revenue loss was $2.3M. The load test, which would have taken 4 hours to write and run, was deprioritized because it didn't fit the sprint scope. Enforcing DoD would have either caught the bottleneck or forced the team to deliver a smaller, load-tested feature that actually stayed up.

- **Smaller teams are better** — 4-6 person teams outperform 8-10 person teams in communication efficiency, decision-making speed, and cohesion. Trade-off: fewer people per team means more teams to manage. Benefit: less overhead, more time building.

- **Use Scrum, don't let Scrum use you** — Adapt the framework to your context. A 1-week sprint for a medical device team may not work. A team doing research might skip the Sprint Review if there's nothing to demo. Trade-off: too much adaptation erodes Scrum benefits. Benefit: context-appropriate practices that the team actually follows.
