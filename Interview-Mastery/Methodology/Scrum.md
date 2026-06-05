# Scrum Study Guide

## 1. Executive Summary

Scrum is a lightweight agile framework for developing, delivering, and sustaining complex products. Based on empiricism (transparency, inspection, adaptation), Scrum uses fixed-length iterations called sprints to deliver incremental value. It defines three roles (Product Owner, Scrum Master, Development Team), five events (Sprint, Sprint Planning, Daily Scrum, Sprint Review, Sprint Retrospective), and three artifacts (Product Backlog, Sprint Backlog, Increment). Scrum is the most widely adopted agile framework globally.

## 2. Core Theory

### 2.1 The Three Pillars of Scrum

- **Transparency**: Significant process aspects must be visible to those responsible
- **Inspection**: Scrum artifacts must be inspected frequently for variances
- **Adaptation**: Process must be adjusted quickly when deviations are detected

### 2.2 Scrum Values

| Value | Description |
|-------|-------------|
| Commitment | Team commits to achieving sprint goals |
| Courage | Team has courage to do the right thing |
| Focus | Team focuses on sprint work |
| Openness | Team is open about work and challenges |
| Respect | Team members respect each other |

### 2.3 Scrum Team Structure

```
+====================================+
|          SCRUM TEAM                |
|  +-----------------------------+  |
|  |   Product Owner (1)         |  |
|  +-----------------------------+  |
|  +-----------------------------+  |
|  |   Scrum Master (1)          |  |
|  +-----------------------------+  |
|  +-----------------------------+  |
|  |   Development Team (3-9)    |  |
|  |   [Cross-functional]        |  |
|  |   [Self-organizing]         |  |
|  +-----------------------------+  |
+====================================+
```

## 3. Under-the-Hood Deep Dive

### 3.1 Sprint Cycle

```
Sprint Planning (max 4 hrs)
       |
       v
Daily Standup (15 min, daily)
       |
       v
Sprint Review (max 4 hrs, end)
       |
       v
Sprint Retrospective (max 3 hrs, end)
       |
       v
Next Sprint Planning
```

### 3.2 Sprint Length

| Duration | Pros | Cons | Best For |
|----------|------|------|----------|
| 1 week | Fast feedback, easy planning | High overhead | Early stage, uncertain |
| 2 weeks | Balance of predictability | Moderate overhead | Most teams |
| 3-4 weeks | Longer to deliver | Slow feedback | Established products |

### 3.3 Artifacts and Commitments

| Artifact | Commitment | Description |
|----------|------------|-------------|
| Product Backlog | Product Goal | Ordered list of everything needed |
| Sprint Backlog | Sprint Goal | Selected items + delivery plan |
| Increment | Definition of Done | Usable product at sprint end |

## 4. Production Code Examples

### 4.1 Sprint Backlog Template

```yaml
sprint_backlog:
  sprint_goal: "Implement user authentication system"
  duration: "2 weeks"
  team_members:
    - name: "Alice"
      capacity: 8
    - name: "Bob"
      capacity: 7
    - name: "Charlie"
      capacity: 10
  total_capacity: 25

  backlog_items:
    - id: US-42
      title: "User login with email/password"
      points: 8
      status: "In Progress"
      owner: "Alice"
      tasks:
        - "Create login form"
        - "Implement authentication API"
        - "Write unit tests"
        - "Add error handling"
      acceptance_criteria:
        - "User can login with email and password"
        - "Invalid credentials show error message"
        - "Session persists for 24 hours"

    - id: US-43
      title: "Password reset flow"
      points: 5
      status: "In Review"
      owner: "Bob"
      tasks:
        - "Create reset password page"
        - "Implement email notification"
        - "Add rate limiting"

    - id: TECH-7
      title: "Upgrade auth library"
      points: 3
      status: "Done"
      owner: "Charlie"
```

### 4.2 Burndown Chart Generator

```python
import matplotlib.pyplot as plt
from datetime import datetime, timedelta

def generate_burndown(sprint_days, total_points, daily_remaining):
    dates = [datetime.now() + timedelta(days=i) for i in range(sprint_days)]
    ideal_line = [total_points * (1 - i/sprint_days) for i in range(sprint_days)]

    plt.figure(figsize=(10, 6))
    plt.plot(dates, ideal_line, '--', label='Ideal', color='gray')
    plt.plot(dates[:len(daily_remaining)], daily_remaining,
             marker='o', label='Actual', color='blue')
    plt.fill_between(dates[:len(daily_remaining)],
                     daily_remaining, ideal_line[:len(daily_remaining)],
                     alpha=0.3,
                     color='red' if daily_remaining[-1] > ideal_line[-1] else 'green')
    plt.title('Sprint Burndown Chart')
    plt.xlabel('Date')
    plt.ylabel('Remaining Points')
    plt.legend()
    plt.grid(True, alpha=0.3)
    return plt
```

### 4.3 Sprint Automation (GitHub Actions)

```yaml
name: Scrum Board Automation

on:
  pull_request:
    types: [opened, closed, ready_for_review]

jobs:
  update-board:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Move to In Review
        if: github.event.action == 'ready_for_review'
        run: |
          curl -X POST https://api.zenhub.com/p1/workspaces/.../repositories/.../issues/${{ github.event.pull_request.number }}/moves \
            -H "X-Authentication-Token: ${{ secrets.ZENHUB_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"pipeline_id": "in_review"}'

      - name: Move to Done
        if: github.event.action == 'closed' && github.event.pull_request.merged == true
        run: |
          curl -X POST https://api.zenhub.com/p1/workspaces/.../repositories/.../issues/${{ github.event.pull_request.number }}/moves \
            -H "X-Authentication-Token: ${{ secrets.ZENHUB_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"pipeline_id": "done"}'
```

### 4.4 Capacity Calculator

```python
def calculate_sprint_capacity(team_members, sprint_days, focus_factor=0.6):
    total_capacity = 0
    for member in team_members:
        available = sprint_days - member['vacation_days'] - member['ceremony_days']
        effective = available * focus_factor
        total_capacity += effective
        member['effective_capacity'] = effective

    story_point_velocity = total_capacity * team_members[0]['points_per_day']

    return {
        'total_person_days': total_capacity,
        'story_point_capacity': story_point_velocity,
        'recommended_backlog': story_point_velocity * 0.8
    }

team = [
    {'name': 'Alice', 'vacation_days': 0, 'ceremony_days': 2, 'points_per_day': 1.5},
    {'name': 'Bob', 'vacation_days': 1, 'ceremony_days': 2, 'points_per_day': 1.2},
    {'name': 'Charlie', 'vacation_days': 0, 'ceremony_days': 2, 'points_per_day': 1.8},
]
print(calculate_sprint_capacity(team, sprint_days=10))
```

## 5. Real-World Scenarios

### 5.1 Rescue: Team Missing Sprint Goals

Symptoms:
- 3 consecutive incomplete sprints
- Team morale declining
- PO frustrated with predictability

Diagnosis:
1. Compare goals to historical velocity
2. Check for unplanned work interruptions
3. Review DoD strictness

Recovery:
1. Reduce sprint to 1 week temporarily
2. Add 20% buffer in planning
3. Enforce no mid-sprint changes
4. Swarm on incomplete items

### 5.2 Distributed Scrum Configuration

```yaml
distributed_team:
  locations:
    - city: "New York"
      timezone: "EST (UTC-5)"
      members: [Alice, Bob]
    - city: "London"
      timezone: "GMT (UTC+0)"
      members: [Charlie, Diana]
    - city: "Bangalore"
      timezone: "IST (UTC+5:30)"
      members: [Eve, Frank]

  agreements:
    core_hours: "14:00-18:00 UTC"
    standup_time: "15:00 UTC"
    async_updates: true
    recording_policy: "All ceremonies recorded"
```

## 6. Performance

### 6.1 Scrum Metrics

```yaml
metrics:
  velocity:
    description: "Points delivered per sprint"
    target: "Stable or increasing trend"

  sprint_goal_success_rate:
    description: "% of sprints achieving goal"
    target: "> 80%"

  predictability:
    description: "Planned vs actual variance"
    target: "< 15% variance"

  happiness_index:
    description: "Team morale (1-5)"
    target: "> 4.0"

  escaped_defects:
    description: "Bugs found in production"
    target: "Decreasing"
```

### 6.2 Sprint Health Assessment

```yaml
sprint_health:
  green:
    - velocity consistent with historical
    - sprint goal achieved
    - no escaped defects
    - happiness >= 4

  yellow:
    - velocity dropped >20%
    - sprint goal partially missed
    - minor defects escaped
    - happiness 3-4

  red:
    - velocity dropped >40%
    - sprint goal not achieved
    - major defects
    - happiness < 3
```

## 7. Security

- Security stories in backlog with priority
- Security review as part of DoD
- Threat modeling in sprint planning
- Penetration testing per release
- Security champion in team

## 8. Common Mistakes

| Mistake | Impact | Solution |
|---------|--------|----------|
| No PO availability | Wrong priorities | PO accessible daily |
| SM also develops | Conflict of interest | Dedicated SM |
| Changing sprint scope | Loss of focus | Protect sprint goal |
| No sprint goal | Aimless sprint | Define goal each sprint |
| Missing retros | No improvement | Make mandatory |
| Standup > 15 min | Waste | Focus on blockers |
| Team too large/small | Inefficiency | 3-9 members |
| No DoD | Technical debt | Create and enforce DoD |
| No estimation history | Inaccurate | Track velocity |
| No backlog refinement | Poor planning | 10% sprint time |

## 9. Senior Engineer Perspective

### 9.1 Scrum Master vs Project Manager

| Activity | Scrum Master | Project Manager |
|----------|-------------|-----------------|
| Governance | Scrum rules | Budget, timeline |
| Role | Servant leader | Manager |
| Focus | Process improvement | Deliverable |
| Decision making | Team-driven | Authority-driven |
| Metrics | Velocity, happiness | Budget, schedule |

### 9.2 When Not to Use Scrum

- Maintenance-only teams (use Kanban)
- Highly uncertain research (use XP/Lean Startup)
- Regulated environments (adapt with compliance gates)
- Very small teams (1-2 people need simpler approach)

## 10. Interview Questions (Easy)

1. What is Scrum?
2. What are the three Scrum roles?
3. What is a sprint?
4. What is a product backlog?
5. What is the daily standup?
6. What is a sprint retrospective?
7. What is the Definition of Done?
8. What is a user story?
9. What is sprint planning?
10. Who owns the product backlog?

## 10. Interview Questions (Medium)

11. Scrum Master vs Product Owner?
12. How to handle unfinished work at sprint end?
13. Velocity vs capacity?
14. How do you estimate work in Scrum?
15. What happens when a team member is on vacation?
16. Multiple teams on same product backlog?
17. Sprint Goal and who defines it?
18. PO changes requirements mid-sprint?
19. Scrum vs SAFe?
20. Measure Scrum team effectiveness?

## 11. Advanced Interview Questions (Hard)

1. Implement Scrum in safety-critical system (aviation, medical).
2. Design Scrum-of-Scrums for 8 teams building one product.
3. Handle technical debt without compromising features.
4. Design early warning dashboard for sprint failure.
5. Adapt Scrum for hardware + software product.
6. Scrum adoption plan for 200-person org.
7. Manage dependencies between Scrum teams.
8. Cross-team retrospective insight sharing.
9. Maintain quality under velocity pressure.
10. Scrum Master rotation program.

## 11. Advanced Interview Questions (System Design)

11. Tool auto-generating retros from git, tickets, CI/CD data.
12. Dependency visualization for multi-team Scrum.
13. Capacity planning from historical Scrum data.
14. Sprint planning optimization algorithm.
15. Scrum Master AI assistant for team health.
16. Cross-team impediment resolution system.
17. Predictive sprint outcome model.
18. Scrum training and certification platform.
19. Real-time Scrum board for distributed teams.
20. Enterprise release coordination with Scrum.

## 12. Expert-Level Interview Questions (Architect)

1. Design enterprise-wide Scrum transformation for 10,000-person financial institution with regulatory compliance, union contracts, 40-year legacy culture.

2. Architect system auto-detecting Scrum anti-patterns across 500+ teams from repo data, communication patterns, delivery metrics.

3. Design compensation system rewarding Scrum values (commitment, courage, focus, openness, respect) over individual heroics.

4. Organization where C-suite operates using Scrum (OKRs as product goals, executive sprints, leadership retros).

5. Framework blending Scrum with Kanban (ops), XP (engineering), Lean Startup (product discovery) into unified system.

6. Contract model enabling true agile partnerships while satisfying procurement, legal, finance.

7. Real-time org health monitoring using Scrum metrics to predict burnout, turnover, productivity decline.

8. Physical/digital workspace for 1000-person Scrum org optimizing flow, collaboration, deep work.

9. System auto-generating improvement experiments from retro insights across thousands of teams.

10. Redesign education to teach Scrum from primary school through university, creating agile-native workers.

## 13. Debugging & Troubleshooting

### 13.1 Scrum Anti-Patterns

| Anti-Pattern | Symptom | Solution |
|-------------|---------|----------|
| Zombie Scrum | Going through motions | Revisit Scrum values |
| Water-Scrum-Fall | Agile ceremonies, waterfall mindset | True cross-functional teams |
| ScrumBut | "We use Scrum but..." | Identify what to change |
| Hero culture | Same person saves every sprint | Spread work, coach others |
| Proxy PO | Delegate makes decisions | PO must be available |

### 13.2 Sprint Recovery

```yaml
mid_sprint_intervention:
  triggers:
    - velocity < 50% of planned at midpoint
    - critical blocker identified
    - team member unavailable
  actions:
    - immediate: "Replan remaining work"
    - short_term: "Remove lowest priority items"
    - escalation: "Inform stakeholders of revised scope"
```

## 14. Comparison Section

### Scrum vs Kanban

| Aspect | Scrum | Kanban |
|--------|-------|--------|
| Cadence | Fixed sprints | Continuous |
| Roles | SM, PO, Dev Team | None prescribed |
| WIP Limits | Implicit (sprint scope) | Explicit |
| Estimation | Required | Optional |
| Changes | No mid-sprint | Anytime |
| Metrics | Velocity, burndown | Cycle time, throughput |
| Best for | Complex products | Support, operations |

### Scrum vs XP

| Aspect | Scrum | XP |
|--------|-------|----|
| Focus | Management process | Engineering practices |
| Practices | Ceremonies, roles | TDD, pair programming, CI |
| Planning | Sprint planning | Release planning |
| Quality | DoD | Built-in quality practices |

## 15. Revision Notes

```
SCRUM SUMMARY
- Framework for complex product delivery
- Based on empiricism (transparency, inspection, adaptation)
- Iterative (sprints) and incremental (increment)

THREE ROLES
- Product Owner: What to build (value)
- Scrum Master: How to work (process)
- Dev Team: Build it (execution)

FIVE EVENTS
- Sprint (container, < 1 month)
- Sprint Planning (what + how)
- Daily Scrum (daily sync)
- Sprint Review (inspect increment)
- Retrospective (inspect process)

THREE ARTIFACTS
- Product Backlog (ordered, evolving)
- Sprint Backlog (planned work)
- Increment (usable product)

THREE COMMITMENTS
- Product Goal (backlog commitment)
- Sprint Goal (sprint commitment)
- Definition of Done (quality commitment)
```

## 16. Cheat Sheet

```text
+======================================================================+
|                     SCRUM CHEAT SHEET                                |
+======================================================================+

  SCRUM FRAMEWORK
+----------------------------------------------------------------------+
| Roles:     Product Owner (1) | Scrum Master (1) | Dev Team (3-9)    |
| Events:    Sprint | Planning | Daily Standup | Review | Retro       |
| Artifacts: Product Backlog | Sprint Backlog | Increment             |
| Commitments: Product Goal | Sprint Goal | Definition of Done       |
+----------------------------------------------------------------------+

  SPRINT PLANNING (max 4 hrs for 2-week sprint)
+----------------------------------------------------------------------+
| Part 1 (What): PO presents top backlog items, team selects           |
| Part 2 (How): Team decomposes into tasks, estimates effort           |
| Output: Sprint Goal + Sprint Backlog                                 |
+----------------------------------------------------------------------+

  DAILY SCRUM (15 min, same time/place)
+----------------------------------------------------------------------+
| Each team member answers:                                            |
| 1. What did I do yesterday?                                          |
| 2. What will I do today?                                             |
| 3. What blockers are in my way?                                      |
| NOT a status report - team sync                                      |
+----------------------------------------------------------------------+

  SPRINT REVIEW (max 4 hrs for 2-week sprint)
+----------------------------------------------------------------------+
| Team demos completed work to stakeholders                            |
| PO discusses what's done and what's not                              |
| Stakeholders give feedback                                           |
| Backlog adjusted based on feedback                                   |
+----------------------------------------------------------------------+

  SPRINT RETROSPECTIVE (max 3 hrs for 2-week sprint)
+----------------------------------------------------------------------+
| Format: Start-Stop-Continue | Sailboat | Mad-Sad-Glad                |
| Focus: What went well? What could improve? Action items             |
| Rule: Blameless, psychological safety required                       |
| Output: At least one actionable improvement                          |
+----------------------------------------------------------------------+

  DEFINITION OF DONE (Example)
+----------------------------------------------------------------------+
| [ ] Code peer reviewed                                               |
| [ ] All tests pass (unit, integration, E2E)                          |
| [ ] Code coverage >= 80%                                             |
| [ ] Acceptance criteria verified                                     |
| [ ] No known P0/P1 defects                                           |
| [ ] Deployed to staging                                              |
| [ ] Smoke tests pass                                                 |
| [ ] Documentation updated                                            |
| [ ] PO approval                                                      |
+----------------------------------------------------------------------+

+======================================================================+
|  PRO TIPS: Sprint Goal is the most important output.                 |
|  Removing impediments is Scrum Master's #1 job.                     |
|  Protect team from interruptions during sprint.                     |
|  Retros must produce actionable outcomes.                           |
|  Self-organization requires trust from management.                  |
|  Done means DONE - releasable, usable, valuable.                    |
+======================================================================+
```
