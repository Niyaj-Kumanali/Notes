# Agile Study Guide

## 1. Executive Summary

Agile is a software development methodology based on iterative development, cross-functional teams, and continuous feedback. Rooted in the Agile Manifesto (2001), it emphasizes individuals and interactions, working software, customer collaboration, and responding to change. Agile encompasses multiple frameworks (Scrum, Kanban, XP, SAFe) that share common values but differ in practices. It has become the dominant approach to software development, adopted by organizations of all sizes worldwide.

## 2. Core Theory

### 2.1 Agile Manifesto - Four Values

1. **Individuals and interactions** over processes and tools
2. **Working software** over comprehensive documentation
3. **Customer collaboration** over contract negotiation
4. **Responding to change** over following a plan

### 2.2 12 Agile Principles

1. Satisfy customer through early/continuous delivery
2. Welcome changing requirements, even late in development
3. Deliver working software frequently (weeks, not months)
4. Business people and developers work together daily
5. Build projects around motivated individuals, trust them
6. Face-to-face conversation is most efficient
7. Working software is primary measure of progress
8. Sustainable development, constant pace
9. Continuous attention to technical excellence
10. Simplicity - maximizing work not done
11. Self-organizing teams produce best architectures
12. Regularly reflect and tune behavior

### 2.3 Agile Methodologies

| Methodology | Key Practices | Best For |
|------------|---------------|----------|
| Scrum | Sprints, standups, retrospectives | Complex product development |
| Kanban | Visualize flow, WIP limits, continuous delivery | Support, maintenance |
| XP (Extreme Programming) | TDD, pair programming, CI | Engineering excellence |
| Lean | Value stream mapping, eliminate waste | Process optimization |
| SAFe | PI planning, ARTs, Lean Portfolio Mgmt | Enterprise scaling |

## 3. Under-the-Hood Deep Dive

### 3.1 Agile vs Waterfall

| Aspect | Agile | Waterfall |
|--------|-------|-----------|
| Requirements | Evolutionary | Fixed upfront |
| Delivery | Incremental | Single release |
| Customer involvement | Continuous | At milestones |
| Team structure | Cross-functional | Siloed |
| Change response | Adaptive | Resistant |
| Documentation | Just enough | Comprehensive |
| Risk | Early visibility | Late discovery |

### 3.2 Estimation Techniques

- **Planning Poker**: Fibonacci sequence (1, 2, 3, 5, 8, 13, 21)
- **T-Shirt Sizing**: XS, S, M, L, XL
- **Affinity Mapping**: Group similar-sized items
- **Dot Voting**: Team votes on complexity
- **Bucket System**: Sort into predefined size buckets

### 3.3 Story Points vs Hours

| Factor | Story Points | Hours |
|--------|-------------|-------|
| Granularity | Coarse (relative) | Fine (absolute) |
| Precision | Low | High |
| Team dependency | Team-specific | Universal |
| Time to estimate | Fast | Slow |
| Useful for | Planning | Scheduling |

## 4. Production Code Examples

### 4.1 Jira Board Configuration

```yaml
board:
  name: "Sprint Board"
  columns:
    - name: Backlog
      status: [Backlog]
      wipLimit: 0
    - name: In Progress
      status: [In Progress, Selected for Dev]
      wipLimit: 5
    - name: In Review
      status: [In Review, Code Review]
      wipLimit: 3
    - name: Done
      status: [Done, Closed]
      wipLimit: 0
  swimlanes:
    - name: Expedite
      wipLimit: 2
    - name: Normal
```

### 4.2 Sprint Planning Template

```markdown
## Sprint Planning: Sprint [N]

**Goal:** [One-line sprint goal]

## Capacity
- Team members: [count]
- Days in sprint: [10]
- Planned velocity: [X] points

## Backlog Items
| ID | Story | Points | Owner |
|----|-------|--------|-------|
| 1  |       |        |       |

## Risks
- Risk 1
- Risk 2
```

### 4.3 Definition of Done

```yaml
definition_of_done:
  code:
    - Code reviewed by peer
    - Linting passes
    - Unit tests pass (coverage >= 80%)
    - No debug statements
  testing:
    - Integration tests pass
    - E2E tests pass
    - Security scan completed
  deployment:
    - Deployed to staging
    - Smoke tests pass
    - Feature flag configured
    - Rollback plan documented
```

### 4.4 Velocity Calculator

```python
def calculate_velocity(historical_sprints):
    points = [s['total_points'] for s in historical_sprints]
    if len(points) < 3:
        return sum(points) / len(points)

    sorted_points = sorted(points)
    trimmed = sorted_points[1:-1]  # Remove outliers

    avg = sum(trimmed) / len(trimmed)
    variance = sum((p - avg) ** 2 for p in trimmed) / len(trimmed)
    std_dev = variance ** 0.5

    return {
        'average': avg,
        'std_dev': std_dev,
        'range_low': avg - std_dev,
        'range_high': avg + std_dev
    }
```

## 5. Real-World Scenarios

### 5.1 Enterprise Agile Adoption

Challenges:
- Resistance to change from traditional teams
- Integrating with existing PMO processes
- Scaling across multiple teams
- Maintaining agile values in bureaucracy

Solutions:
- Executive sponsorship and agile champions
- Start with pilot teams, expand gradually
- Use SAFe or LeSS for scaling
- Adapt ceremonies to org culture

### 5.2 Distributed Agile Teams

Best Practices:
- Overlap working hours by 4+ hours
- Use async communication for updates
- Record ceremonies for absent members
- Rotate meeting times across timezones
- Invest in video conferencing tools
- Quarterly face-to-face gatherings

## 6. Performance

### 6.1 Agile Metrics

| Metric | What It Measures | Target |
|--------|-----------------|--------|
| Velocity | Points per sprint | Consistent trend |
| Cycle Time | Start to finish | Decreasing |
| Lead Time | Request to delivery | Decreasing |
| WIP | Work in progress | < team size |
| Throughput | Items per sprint | Increasing |
| Burndown | Work remaining vs time | On track |

## 7. Security

### 7.1 Agile Security Practices

- Security requirements in user stories
- Threat modeling in sprint planning
- Security review as definition of done
- Penetration testing per release
- Security training for all team members

### 7.2 DevSecOps in Agile

- **Planning**: Security stories prioritized
- **Daily standup**: Security blockers raised
- **Review**: Security demo included
- **Retro**: Security improvements identified

## 8. Common Mistakes

| Mistake | Impact | Solution |
|---------|--------|----------|
| Standups become status reports | Waste of time | Focus on blockers |
| Retros without action | No improvement | Track action items |
| Story points as performance metric | Gaming the system | Planning only |
| Too many WIP items | Context switching | Enforce WIP limits |
| Mid-sprint changes | Loss of focus | Protect sprint goal |
| No DoD | Technical debt | Create and enforce DoD |
| PO unavailable | Wrong priorities | PO must be available |
| Team too large (over 9) | Communication overhead | Split team |
| No automated testing | Slow regression | CI/CD investment |

## 9. Senior Engineer Perspective

### 9.1 Scaling Frameworks

- **SAFe**: 50+ teams in large enterprises
- **LeSS**: 3-8 teams on same product
- **Nexus**: 3-9 teams, Scrum.org framework
- **Spotify Model**: Tribes, squads, chapters, guilds

### 9.2 When Agile Fails

1. No executive buy-in for cultural change
2. Teams not truly cross-functional
3. Technical debt prevents rapid iteration
4. Org structure conflicts with agile values
5. Too much process, not enough agility

## 10. Interview Questions (Easy)

1. What is Agile software development?
2. What are the four values of the Agile Manifesto?
3. What is a sprint in Scrum?
4. What is a user story?
5. What is a daily standup?
6. What is a product backlog?
7. Agile vs Waterfall?
8. What is a sprint retrospective?
9. What is velocity?
10. What is the role of a Product Owner?

## 10. Interview Questions (Medium)

11. Scrum vs Kanban differences?
12. How do you estimate user stories?
13. What is technical debt and how to manage it?
14. How to handle changing requirements mid-sprint?
15. What is Definition of Done and why important?
16. How to scale Agile across multiple teams?
17. What Agile metrics matter?
18. What is a spike?
19. How to handle poorly estimated stories?
20. Velocity vs capacity?

## 11. Advanced Interview Questions (Hard)

1. Implement Agile in regulated industry (finance, healthcare).
2. Design scaling strategy for 500-person org.
3. Balance technical debt with feature delivery.
4. Design metrics system that prevents gaming.
5. Handle team missing sprint commitments consistently.
6. Design Agile adoption roadmap for waterfall org.
7. Integrate UX design into Agile sprints.
8. Distributed Agile across 10 timezones.
9. Maintain architectural vision in Agile.
10. Continuous improvement program from retro data.

## 11. Advanced Interview Questions (System Design)

11. Agile portfolio management for 200+ teams.
12. Automated Agile estimation from historical data.
13. Real-time Agile analytics dashboard.
14. Dependency management for multi-team Agile.
15. Agile coaching platform for 1000+ teams.
16. Automated retrospective analysis system.
17. Predict sprint success from historical patterns.
18. Cross-team coordination for large-scale Agile.
19. Agile transformation measurement system.
20. Feature flag management with Agile delivery.

## 12. Expert-Level Interview Questions (Architect)

1. Design enterprise Agile transformation for 10,000-person org across 50 countries with regulatory requirements.

2. Architect measurement system correlating Agile practices with business outcomes (revenue, customer satisfaction, time-to-market).

3. Design system auto-identifying cross-team dependencies, bottlenecks, coordination failures across 200+ teams.

4. Learning organization model where retro insights from 1000+ teams are captured, analyzed, redistributed.

5. Value stream management platform connecting strategic objectives to team backlogs with alignment verification.

6. Compensation system rewarding agile behaviors (collaboration, learning, adaptability) over individual metrics.

7. AI-assisted sprint planning optimizing story selection based on capacity, dependencies, risk, value.

8. Transparency platform providing stakeholder visibility without creating surveillance culture.

9. Contract model enabling true agile partnerships in fixed-budget, fixed-scope corporate environment.

10. Redesign physical/digital workplace maximizing flow, collaboration, creativity in hybrid world.

## 13. Debugging & Troubleshooting

### 13.1 Common Anti-Patterns

| Symptom | Diagnosis | Fix |
|---------|-----------|-----|
| Standups 30+ min | Status reporting | Focus on blockers |
| Velocity drops | Overcommitment | Historical data planning |
| Same retro issues | No follow-through | Track action items |
| PO is bottleneck | PO overloaded | APIO role |

### 13.2 Sprint Rescue

```yaml
critical_sprint_fixes:
  scope_reduction:
    - Identify minimum viable sprint goal
    - Move non-critical stories back
  team_intervention:
    - Remove interruptions
    - Swarm on critical stories
    - Pair programming
  blocker_escalation:
    - Find root cause
    - Escalate to management
```

## 14. Comparison Section

### Scrum vs Kanban

| Aspect | Scrum | Kanban |
|--------|-------|--------|
| Cadence | Fixed sprints | Continuous flow |
| Roles | SM, PO, Dev Team | None prescribed |
| Estimation | Required | Optional |
| WIP Limits | Implicit (sprint scope) | Explicit |
| Changes | No mid-sprint | Anytime |
| Metrics | Velocity, burndown | Cycle time, throughput |
| Best for | Complex products | Support, operations |

### Agile vs Waterfall

| Aspect | Agile | Waterfall |
|--------|-------|-----------|
| Approach | Iterative, incremental | Sequential, phase-gate |
| Requirements | Emergent | Fully defined upfront |
| Customer | Continuous involvement | At milestones |
| Risk | Early discovery | Late discovery |
| Adaptability | High | Low |

## 15. Revision Notes

```
AGILE MANIFESTO (4 Values)
1. Individuals & interactions > Processes & tools
2. Working software > Comprehensive documentation
3. Customer collaboration > Contract negotiation
4. Responding to change > Following a plan

12 PRINCIPLES (Summary)
- Customer satisfaction, welcome changes, deliver frequently
- Business + dev daily, motivated teams, face-to-face
- Working software = progress, sustainable pace
- Technical excellence, simplicity, self-organizing teams
- Reflect and adjust regularly

COMMON CEREMONIES
- Sprint Planning (2 hrs/week)
- Daily Standup (15 min)
- Sprint Review (1 hr/week)
- Retrospective (1 hr/week)
- Backlog Refinement (10% sprint)

KEY ROLES
- Product Owner: Value maximizer
- Scrum Master: Process guardian
- Development Team: Builders
```

## 16. Cheat Sheet

```text
+======================================================================+
|                     AGILE CHEAT SHEET                                |
+======================================================================+

  AGILE MANIFESTO VALUES
+----------------------------------------------------------------------+
| Individuals & Interactions  >  Processes & Tools                     |
| Working Software            >  Comprehensive Documentation           |
| Customer Collaboration      >  Contract Negotiation                  |
| Responding to Change        >  Following a Plan                      |
+----------------------------------------------------------------------+

  SCRUM ROLES
+----------------------------------------------------------------------+
| Product Owner  | Maximizes value, manages backlog                    |
| Scrum Master   | Coaches team, removes impediments                  |
| Dev Team       | Self-organizing, cross-functional, 3-9 members     |
+----------------------------------------------------------------------+

  SCRUM CEREMONIES
+----------------------------------------------------------------------+
| Sprint Planning     | What + How for sprint (max 4 hrs/2wk)          |
| Daily Standup       | Sync + plan next 24h (15 min)                 |
| Sprint Review       | Demo completed work to stakeholders           |
| Retrospective       | Inspect + adapt team process                  |
| Backlog Refinement  | Keep backlog groomed (10% sprint time)         |
+----------------------------------------------------------------------+

  KANBAN PRINCIPLES
+----------------------------------------------------------------------+
| 1. Visualize the workflow                                            |
| 2. Limit Work In Progress (WIP)                                      |
| 3. Manage flow                                                       |
| 4. Make process policies explicit                                    |
| 5. Implement feedback loops                                          |
| 6. Improve collaboratively, evolve experimentally                   |
+----------------------------------------------------------------------+

  AGILE ESTIMATION
+----------------------------------------------------------------------+
| Story Points: Relative sizing (Fibonacci: 1, 2, 3, 5, 8, 13, 21)   |
| Planning Poker: Team estimates simultaneously                        |
| T-Shirt Sizes: XS, S, M, L, XL                                       |
| Affinity Mapping: Group similar items                                |
+----------------------------------------------------------------------+

  DEFINITION OF DONE CHECKLIST
+----------------------------------------------------------------------+
| [ ] Code reviewed                                                    |
| [ ] Unit tests pass & meet coverage threshold                        |
| [ ] Integration tests pass                                           |
| [ ] Acceptance criteria met                                          |
| [ ] Documentation updated (if needed)                               |
| [ ] Deployed to staging                                              |
| [ ] Smoke tests pass                                                 |
| [ ] PO approved                                                      |
+----------------------------------------------------------------------+

+======================================================================+
|  PRO TIPS: Protect the sprint goal. Keep retros blameless.           |
|  Velocity is for planning, NOT performance evaluation.              |
|  WIP limits are the most impactful Kanban practice.                 |
|  Invest in automated testing for sustainable pace.                  |
|  Self-organizing teams need trust, not control.                     |
+======================================================================+
```
