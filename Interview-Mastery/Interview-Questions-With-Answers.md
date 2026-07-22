# Interview Questions Bank

This file follows the exact same structure and numbering as `Interview-Questions-Bank.md`. Each question has a direct answer and an `If asked more` expansion below it.

---

## HR Questions

1. Tell me about yourself.
   - **Answer:** My name is Niyaj Kumanali. I am a Software Engineer with around 2.5+ years of experience, mainly in backend development, database optimization, and real-time data processing. I work with Java, Spring Boot, MSSQL, Kafka, AWS, Redis, Docker, Jenkins, and React. In my current role at Talentpace, I worked on Lenovo India projects and a cold-chain IoT monitoring system, where my focus was performance, reliability, and data correctness.
   - **If asked more:** I'd walk through my career step by step — my BE in CS, joining Talentpace as a fresher, and the projects I grew through. When I start with CDMS, I explain the business context: Lenovo needed channel partner reports that were taking 5 hours to generate. I took ownership of optimizing the stored procedures, analyzed execution plans, fixed indexes and rewritten queries, and brought it down to 12 minutes. For the inventory system, I'd describe how we solved reconciliation across 200+ partners with a centralized MSSQL validation engine. And for cold-chain, I'd walk the full IoT pipeline from MQTT gateways through Kafka to Grafana dashboards.
2. Walk me through your resume.
   - **Answer:** I completed BE in Computer Science and then joined Talentpace Pvt Ltd as a Software Engineer. My resume mainly shows backend work, SQL Server optimization, Kafka-based pipelines, AWS services, and React dashboards. The strongest projects are Lenovo CDMS, Real-Time Partner Inventory, and Cold-Chain Monitoring.
   - **If asked more:** I'd expand on each project in the order they appear on my resume. Starting with CDMS because it was my earliest impact project — I debugged a 5-hour stored procedure bottleneck and optimized it to 12 minutes. Then the inventory validation engine where I handled 10,000+ serial records across 200+ channel partners. And the cold-chain IoT system where I worked across the full stack from MQTT ingestion to React dashboards.
3. Why are you looking for a change?
   - **Answer:** I am looking for a change mainly for career growth and better technical exposure. I have learned a lot in my current role, and now I want to work on larger backend systems, improve in microservices and system design, and take more ownership.
   - **If asked more:** At Talentpace, I've had good opportunities to work on performance tuning, real-time pipelines, and full-stack features. But I've reached a point where I want to work on systems with higher scale — more users, more data, more services. I want to be part of a team where I can learn from senior engineers on system design, distributed architecture, and production operations at scale.
4. Why do you want to join our company?
   - **Answer:** I want to join because this role matches my backend experience and the type of work I want to continue doing. I am looking for good technical challenges, ownership, and a team where I can contribute using Java, Spring Boot, databases, Kafka, and cloud.
   - **If asked more:** I've looked at your tech stack and it aligns well with what I work on daily — Java, Spring Boot, and distributed systems. More importantly, I want to work on products where my backend work directly impacts users. I've read about your engineering challenges around scalability and data processing, and those are exactly the problems I want to solve.
5. Why should we hire you?
   - **Answer:** You should hire me because I have practical production experience in backend development, SQL optimization, and real-time systems. I have delivered measurable impact, like reducing stored procedure execution from around 5 hours to under 12 minutes and improving inventory validation across 200+ channels.
   - **If asked more:** Beyond the numbers, I bring a problem-solving mindset — I don't just implement features, I understand the business context, find the root cause of issues, and deliver solutions that work in production. For CDMS, I didn't just optimize queries; I validated that the optimized procedures returned the same correct results as the originals. For the cold-chain project, I made sure the system handled real-world edge cases like network failures and sensor data bursts.
6. What are your strengths?
   - **Answer:** My main strengths are problem-solving, ownership, and backend debugging. I try to understand the root cause before changing code, especially in database and performance issues.
   - **If asked more:** For example, in the CDMS project, the stored procedures were taking 5 hours but nobody knew why. Instead of randomly rewriting everything, I started with the actual execution plan, identified that a missing index and a non-sargable WHERE clause were the primary bottlenecks, fixed those specific issues, and brought runtime down to 12 minutes. I take the same approach to any problem — understand first, then fix.
7. What is your weakness?
   - **Answer:** Earlier I used to spend extra time trying to make a solution perfect before sharing it. Now I break work into smaller parts, discuss early, and take feedback faster.
   - **If asked more:** I realized this was a weakness when I spent two days perfecting a query optimization but the business need had shifted. Now I share my approach early — I'll say "here's what I'm thinking, here's the estimated improvement, let me know if this direction works" — and adjust based on feedback. It saves time and keeps the team aligned.
8. Where do you see yourself in the next 3 to 5 years?
   - **Answer:** In the next 3 to 5 years, I want to become a strong backend engineer who can design and own scalable systems end to end, and also guide juniors when needed.
   - **If asked more:** Specifically, I want to move beyond writing code to owning system design decisions — choosing the right architecture, balancing trade-offs, and ensuring reliability. I also want to contribute to code reviews and mentoring, because explaining concepts to others has helped me solidify my own understanding in my current role.
9. What motivates you at work?
   - **Answer:** I am motivated when my work creates clear impact. For example, improving performance, reducing manual effort, or making real-time systems more reliable gives me confidence that my work is useful.
   - **If asked more:** The inventory validation engine is a good example — before our system, the Lenovo finance team spent a week every month manually reconciling partner data. After we built the automated validation engine, that process became 15 seconds with clear audit trails. Knowing my code saved a team a week of manual work every month is what keeps me motivated.
10. What kind of work environment do you prefer?
    - **Answer:** I prefer an environment with clear communication, ownership, code reviews, and good technical learning. I am comfortable with Agile/Scrum and cross-functional collaboration.
    - **If asked more:** In my current team, we have daily stand-ups, sprint planning, and code reviews for every PR. I've found that good code reviews are where I learn the most — both from receiving feedback and from reviewing others' code. I also value clear requirements; I'd rather ask clarifying questions upfront than build the wrong thing.
11. Are you comfortable working in a team?
    - **Answer:** Yes, I am comfortable working in a team. In my current role, I worked with backend, frontend, QA, and operations teams, and I try to communicate clearly and take feedback positively.
    - **If asked more:** In the cold-chain project, I worked closely with the frontend team to define the API contracts — we'd agree on response formats in Swagger before either side started coding. With QA, I helped write test scenarios for edge cases like sensor disconnections. With operations, I documented the deployment steps for Jenkins. I think good teamwork is about communication and respect for each person's expertise.
12. Are you comfortable working independently?
    - **Answer:** Yes, I am comfortable working independently. Once the requirement is clear, I can analyze, implement, test, and update the team. If something is unclear, I discuss early.
    - **If asked more:** The CDMS optimization was largely an independent effort — I owned the analysis, the changes, and the validation. I communicated with my lead on progress and blockers, but the day-to-day work was self-driven. I like that balance: independence to go deep on a problem, but enough team communication to stay aligned.
13. How do you handle pressure?
    - **Answer:** I handle pressure by identifying the priority, breaking the work into smaller steps, and focusing on the most critical part first. I also keep the team updated about risks or blockers.
    - **If asked more:** During the CDMS optimization, the pressure was high because the nightly batch had been failing for weeks and the business team needed their reports. I didn't try to fix everything at once — I identified the single biggest bottleneck (a non-sargable query causing a full table scan), fixed it first, and communicated the improvement. Then I iterated on the next bottleneck. This approach kept things moving without introducing risk.
14. How do you handle tight deadlines?
    - **Answer:** For tight deadlines, I focus on must-have items first, keep the implementation simple, test important flows properly, and communicate early if something needs more time.
    - **If asked more:** In the inventory project, the business team needed the validation engine ready before the quarterly partner filing deadline. I prioritized the core validation rules first — the 80% case that handled most records — and pushed advanced edge-case handling to a follow-up sprint. I communicated this trade-off clearly so the team could set expectations with stakeholders.
15. How do you prioritize tasks?
    - **Answer:** I prioritize based on business impact, urgency, dependency, and risk. Production issues or blockers for other team members come first.
    - **If asked more:** For example, when the cold-chain Kafka consumer lag was growing during peak hours, that became the top priority over any feature work because it affected real-time monitoring. I fixed the consumer concurrency issue first, then went back to the feature I was building. I always ask: what breaks if this isn't done today?
16. Tell me about a time you handled a difficult situation.
    - **Answer:** One difficult situation was optimizing Lenovo CDMS stored procedures that were taking almost 5 hours. I analyzed execution plans, optimized queries and indexes, and reduced execution time to under 12 minutes without breaking report correctness.
    - **If asked more:** The hardest part wasn't the technical optimization — it was the pressure. The nightly batch had been failing for weeks, and every day without reports meant delayed partner incentives. I took a methodical approach: captured the actual execution plan, found the most expensive operators, fixed them one at a time, and validated correctness after each change by comparing output. After each change I re-ran the procedure to measure improvement. It took discipline to not rewrite everything at once.
17. Tell me about a time you made a mistake.
    - **Answer:** Earlier I did not always document the reason behind technical changes clearly. I improved by explaining before-and-after impact and adding useful comments or review notes where needed.
    - **If asked more:** In the CDMS project, I optimized a query but didn't document why I changed a JOIN from LEFT to INNER. Three months later, another developer was confused about whether it was intentional. Now I add comments explaining the reasoning — like "changed to INNER JOIN because the filter in WHERE already excludes NULLs, and INNER is 40% faster according to the execution plan."
18. Tell me about a time you received critical feedback.
    - **Answer:** I take feedback positively. One feedback I received was to communicate my reasoning better during reviews, so I started sharing more context for technical and performance-related changes.
    - **If asked more:** A senior dev once reviewed my PR and pointed out that I had made several performance optimizations but hadn't explained the reasoning in the PR description. He was right — the changes looked arbitrary without context. Now I always include "why" in PR descriptions: "changed this join because the execution plan showed a table scan, and adding index X eliminated it."
19. Tell me about a time you disagreed with a teammate.
    - **Answer:** If I disagree with a teammate, I discuss based on facts like maintainability, performance, testing effort, and risk. I am open to changing my view if the other approach is better.
    - **If asked more:** In the cold-chain project, a teammate wanted to store all raw sensor data in MSSQL for simplicity. I disagreed because time-series data grows fast and MSSQL wasn't optimized for that workload. I explained that InfluxDB would give us better compression, faster time-range queries, and automatic downsampling. We did a small proof of concept, and the results showed InfluxDB used 70% less storage and queries were 10x faster. We went with InfluxDB.
20. Tell me about a time you helped a teammate.
    - **Answer:** I have helped teammates understand existing backend logic, database queries, and project flow. I try to explain the issue clearly so they can continue independently.
    - **If asked more:** A new team member was struggling with the CDMS stored procedure logic — 400 lines of complex SQL with multiple CTEs. I sat with them, walked through the procedure section by section, and drew the data flow on a whiteboard. After that session, they could make changes independently. I also created a documentation page explaining each CTE's purpose, which helped the whole team.
21. Tell me about a time you learned something quickly.
    - **Answer:** In the cold-chain project, I had to learn AWS IoT Core, Kafka, InfluxDB, Grafana, and the real-time data flow quickly. I learned step by step by following the data from gateways to dashboards.
    - **If asked more:** I started by setting up a single MQTT publisher on my laptop sending test data to AWS IoT Core. Once that worked, I added Lambda to receive and transform the message. Then I configured a local Kafka broker and Spring Boot consumer. I built the system incrementally, understanding each component before adding the next. Within two weeks, I had a working end-to-end pipeline.
22. Tell me about a time you took ownership of a task.
    - **Answer:** In the CDMS optimization work, I took ownership of analyzing slow stored procedures, improving them, and validating both performance and report correctness.
    - **If asked more:** Nobody asked me to optimize the procedures — the team had accepted the 5-hour runtime as normal. I raised the issue, showed the execution plan analysis, and volunteered to fix it. I set up a comparison framework to validate output before and after changes, kept a log of every optimization and its impact, and presented the results to the team. This ownership mindset is something I bring to every task.
23. Tell me about a time you improved an existing system.
    - **Answer:** I improved the Lenovo CDMS system by optimizing stored procedures and reducing execution time from around 5 hours to under 12 minutes.
    - **If asked more:** The system had been running for years with the same stored procedures. The slow performance was accepted as normal because "that's just how long it takes." I analyzed the execution plan, found that a missing composite index was causing table scans on a 5-million-row table, and that a non-sargable WHERE clause was blocking index usage. Fixing those two things alone cut runtime by 80%. Then I iterated on the remaining bottlenecks.
24. Tell me about a time you worked with unclear requirements.
    - **Answer:** When requirements are unclear, I separate confirmed points from assumptions and ask specific questions about input, output, validations, and edge cases.
    - **If asked more:** In the inventory project, the business team said they wanted "partner inventory validation" but didn't have detailed rules. I created a document with questions: what constitutes a valid serial number, what should happen with duplicates, how to handle records for inactive partners, what the output format should be. I walked through it with the business analyst and got clear answers before writing code. It saved us from building the wrong thing.
25. Tell me about a time you handled production pressure.
    - **Answer:** Under production pressure, I focus on root cause, controlled changes, validation, and communication. I avoid random fixes, especially for database or API issues.
    - **If asked more:** When the cold-chain Kafka consumer lag was spiking to 500,000 messages, the dashboard was showing data 4 hours old. Instead of randomly tuning Kafka configs, I checked the consumer logs and found InfluxDB was throwing "partial write" errors. The root cause was that the consumer was single-threaded and couldn't keep up. I increased consumer concurrency and added exponential backoff for InfluxDB writes. Lag dropped to near zero within 10 minutes.
26. What are your short-term goals?
    - **Answer:** My short-term goal is to work in a backend-focused role where I can contribute using Java, Spring Boot, SQL, Kafka, and AWS, while improving system design.
    - **If asked more:** I want to deepen my understanding of distributed systems — how to design services that are resilient, scalable, and observable. In my current projects, I've worked with Kafka and microservices patterns, but I want to move from implementing features to making architectural decisions: choosing between sync and async communication, designing for fault tolerance, and setting up proper monitoring.
27. What are your long-term goals?
    - **Answer:** My long-term goal is to become a senior backend engineer or technical lead who can design scalable systems and own critical services.
    - **If asked more:** I want to reach a point where I can take a business requirement, design the system architecture, break it into services, define contracts, and guide the team through implementation. I'm building towards this by learning system design patterns, practicing incident response, and taking ownership of cross-cutting concerns like monitoring and CI/CD in my current projects.
28. What do you expect from your manager?
    - **Answer:** I expect clear communication, useful feedback, and guidance on priorities. I also value ownership and technical growth.
    - **If asked more:** I appreciate managers who give me space to solve problems independently but step in when I'm stuck or when priorities need adjustment. Regular one-on-ones with honest feedback about what I'm doing well and where I can improve are important for my growth. I also value managers who advocate for their team's technical growth — whether it's training, conference budgets, or challenging project assignments.
29. What do you expect from your team?
    - **Answer:** I expect collaboration, clear communication, and a good code review culture where people share knowledge and solve problems together.
    - **If asked more:** A good team for me is one where people review each other's code rigorously but respectfully, where knowledge is shared openly, and where people are willing to help when someone is stuck. I've learned a lot from code reviews in my current team, and I want to continue in an environment where that's the norm rather than the exception.
30. What makes you different from other candidates?
    - **Answer:** My practical production experience makes me different. I have worked on performance optimization, real-time pipelines, and business-critical validation systems with measurable results.
    - **If asked more:** I don't just list technologies on my resume — I've used them in production to solve real business problems. The CDMS optimization saved a team weeks of manual work. The inventory engine reduced reconciliation from days to seconds. The cold-chain system reduced temperature excursions by 85%. I bring both the technical depth and the business impact mindset.
31. What are your salary expectations?
    - **Answer:** I am open to discussing compensation based on the role, responsibilities, and company standards. My main focus is the right opportunity and technical growth.
    - **If asked more:** I've done market research and I understand the range for my experience level. I'm looking for a role that offers fair compensation along with good technical challenges and growth opportunities. I'm happy to discuss this further once we both agree this is a good fit.
32. What is your notice period?
    - **Answer:** My notice period is as per my current company policy. I can discuss the exact timeline based on the offer process and joining requirement.
    - **If asked more:** Typically it's 30 days, but I can negotiate based on the joining timeline and project handover needs. I'd be transparent about the exact date once we have an offer.
33. Are you open to relocation?
    - **Answer:** Yes, I am open to relocation depending on the role, location, and overall opportunity.
    - **If asked more:** I'm flexible on location if the role and team are the right fit. For the right opportunity, I'm willing to relocate. I'd like to understand the work-from-home policy and office expectations as well.
34. Are you comfortable with hybrid or work-from-office?
    - **Answer:** Yes, I am comfortable with hybrid or work-from-office based on company policy and team requirement.
    - **If asked more:** In my current role, I've worked both remotely and from the office. I'm productive in either setup. What matters most is clear communication and collaboration, whether in person or over Slack/Teams.
35. Are you comfortable working in shifts if required?
    - **Answer:** I am comfortable if occasional support or planned shift work is required. I would like to understand the exact expectations clearly.
    - **If asked more:** I've handled production issues outside regular hours during critical incidents, so occasional shift work is fine. For regular shifts, I'd like to understand the schedule and how the team handles handovers.
36. What do you know about our company?
    - **Answer:** From what I understand, this role is aligned with backend development and scalable systems. I would like to know more about the team, product, and current engineering challenges.
    - **If asked more:** I've done some research on your company. I know the products you build and the tech stack you use. What I'm most interested in is understanding the current engineering challenges — what are the scaling bottlenecks, what's the team structure, and what would a typical day look like in this role.
37. What do you know about this role?
    - **Answer:** Based on the role, it needs backend development, problem-solving, API development, database knowledge, and production system experience, which matches my background.
    - **If asked more:** From the job description, this role involves building and maintaining backend services, working with databases, and collaborating with frontend teams. My experience with Spring Boot, MSSQL/PostgreSQL, Kafka, and AWS maps well to these requirements. I'd love to hear more about the specific projects the team is working on.
38. What type of projects do you want to work on?
    - **Answer:** I want to work on backend-heavy projects where scalability, performance, data correctness, and reliability are important.
    - **If asked more:** I enjoy projects where I can go deep into the backend — optimizing database queries, designing data pipelines, building robust APIs, and ensuring system reliability. I'm less interested in pure frontend work. The cold-chain project was great because it had backend complexity: real-time data, Kafka pipelines, time-series databases, and monitoring — that's my sweet spot.
39. What is your preferred technology stack?
    - **Answer:** My preferred stack is Java, Spring Boot, SQL databases, Kafka, Redis, AWS, Docker, Jenkins, and React when frontend work is required.
    - **If asked more:** I'm strongest in Java with Spring Boot, and I've used MSSQL and PostgreSQL extensively. For real-time data, I've worked with Kafka and Redis. For cloud, I've used EC2, IoT Core, Lambda, and S3. I'm comfortable in this stack but also open to learning new technologies if the project needs them.
40. Do you prefer backend or full-stack development?
    - **Answer:** My core strength is backend development. I can work on React when needed, but I enjoy APIs, databases, Kafka pipelines, and performance tuning more.
    - **If asked more:** I've built React dashboards for the cold-chain project, so I can handle frontend work, but my passion is backend engineering — designing data models, optimizing queries, building reliable APIs, and working on data pipelines. Given a choice, I'd pick backend problems every time.
41. Why did you choose software development?
    - **Answer:** I chose software development because I like solving logical problems and building systems that solve real business problems.
    - **If asked more:** In college, I enjoyed programming because I could build something from nothing and see it work. Professionally, I love that my code has real impact — when the inventory validation engine I built saves a finance team a week of manual work every month, that's deeply satisfying. It's the combination of logic, creativity, and impact that keeps me in this field.
42. What was your biggest learning in your current company?
    - **Answer:** My biggest learning is that production systems need more than working code. Performance, monitoring, validation, maintainability, and business impact are equally important.
    - **If asked more:** When I started, I thought if the code compiled and passed tests, the job was done. Experience taught me otherwise. The CDMS project taught me that a query that works on 1,000 rows might fail on 1 million rows. The cold-chain project taught me that monitoring and alerting are as important as the data pipeline itself — without proper lag monitoring, we wouldn't have caught the Kafka consumer issue until the dashboard was hours behind. I now think about production readiness from day one.
43. What is your biggest achievement so far?
    - **Answer:** My biggest achievement is optimizing Lenovo CDMS stored procedures and reducing execution time from around 5 hours to under 12 minutes.
    - **If asked more:** Beyond the technical achievement, the impact was significant. The business team could now get their reports the same day instead of waiting overnight. Partners received their incentives on time because the data was available earlier. It also changed how the team approached performance — we started adding execution plan reviews as part of our development process.
44. What is your biggest challenge so far?
    - **Answer:** My biggest challenge was improving performance in an existing reporting system without breaking correctness. It needed careful execution plan analysis, query rewriting, indexing, and validation.
    - **If asked more:** The scariest part was making changes to a system that directly affected partner incentive payments. An incorrect optimization could mean a partner was underpaid or overpaid, with real financial consequences. That's why I validated every change by comparing output — I wrote a script that ran the old and new queries, compared results row by row, and flagged any differences. Only when there were zero differences did I deploy the change.
45. How do you keep yourself updated technically?
    - **Answer:** I keep myself updated by reading documentation, revising concepts, practicing from real project issues, and learning from code reviews.
    - **If asked more:** I follow Java and Spring release notes to stay current with new features. When I face a problem I haven't solved before, I research it thoroughly — reading official docs, blog posts, and Stack Overflow. I also learn a lot from code reviews: when a senior dev suggests a better approach, I understand why it's better and apply that learning going forward.
46. How do you handle repetitive tasks?
    - **Answer:** If a task is repetitive, I complete it carefully first and then check whether it can be automated or standardized.
    - **If asked more:** In the CDMS project, we had to compare old and new procedure outputs manually for validation. I wrote a script that automated the comparison — it ran both versions, highlighted differences, and generated a report. This reduced validation time from an hour to a few minutes and eliminated human error. I try to apply this mindset to any repetitive task.
47. How do you handle ambiguity?
    - **Answer:** I handle ambiguity by asking specific questions, identifying assumptions, and confirming expected input, output, and edge cases early.
    - **If asked more:** In the inventory project, the requirements started as "we need to validate partner data." I broke that down into concrete questions: what fields need validation, what are the valid values, what should happen with invalid data, who needs to be notified. I created a requirements document and got sign-off before writing code. This approach has saved me from building wrong features multiple times.
48. What would your current team say about you?
    - **Answer:** My current team would say I take ownership, debug backend and database issues well, and support teammates when needed.
    - **If asked more:** I think they'd also say I'm thorough — I don't just fix the surface issue, I investigate root causes. When the Kafka consumer lag spiked, I didn't just restart the consumer; I found that the concurrency setting was wrong and fixed it permanently. They'd also say I'm approachable when someone needs help with SQL or debugging a backend issue.
49. What are you expecting from your next role?
    - **Answer:** I expect a backend-focused role where I can work on scalable systems, improve design skills, and contribute to meaningful production applications.
    - **If asked more:** I want a role where I can own significant backend components — designing APIs, building data pipelines, improving performance. I'm looking for a team that values engineering excellence: code reviews, testing, monitoring, and continuous improvement. And I want to work on products where my work has visible impact on users.
50. Do you have any questions for us?
    - **Answer:** Yes. I would like to know about the backend systems, team expectations, technology stack, code review process, deployment process, and ownership model.
    - **If asked more:** Specifically: what does a typical sprint look like? How does the team handle on-call and incidents? What's the balance between new feature work and technical debt reduction? How does the team make architectural decisions? And what would success look like for this role in the first 90 days?

---

## Resume and Project Questions

1. Explain your current role at Talentpace Pvt Ltd.
   - **Answer:** At Talentpace, I work as a Software Engineer where I own backend development end-to-end — from designing REST APIs in Spring Boot to optimizing MSSQL queries that were blocking business reporting. I also integrate real-time data pipelines using Kafka and AWS, and I collaborate closely with QA and frontend teams to ensure our deployments are smooth and our APIs are well-documented with Swagger. My focus is always on writing code that is maintainable, secure, and performant under load.
   - **If asked more:** I'd pick the inventory validation engine and explain how I designed the validation rules in MSSQL, why I chose set-based operations over cursors, and how that decision directly drove the 15-second processing time.
2. What are your main responsibilities in your current project?
   - **Answer:** I develop and maintain Spring Boot microservices that serve as the backbone for our partner management and IoT monitoring platforms. I spend a significant amount of time analyzing MSSQL execution plans and rewriting stored procedures to eliminate bottlenecks — for instance, I once cut a procedure from 5 hours to 12 minutes just by fixing a nested loop join that should have been a hash match. I also handle API security with Spring Security and JWT, write validation logic for incoming data streams, and set up CI/CD pipelines in Jenkins so that deployments are automated and consistent.
   - **If asked more:** I start with the slowest query in the pipeline, capture its actual execution plan, identify the highest-cost operator, and only then decide whether an index change, a query rewrite, or a schema adjustment is needed.
3. Which project from your resume are you most confident about?
   - **Answer:** I'm most confident about the Lenovo CDMS project because it pushed me deep into MSSQL internals — I wasn't just writing queries, I was reading actual execution plans, understanding row estimates versus actuals, and making surgical changes that had a massive impact. Taking a batch process from 5 hours to under 12 minutes taught me that database optimization is less about guessing and more about systematic diagnosis: find the scan, fix the join, add the missing index, verify correctness, then move to the next bottleneck.
   - **If asked more:** One specific execution plan anomaly I found was a key lookup causing 4.8 million logical reads on a 50k-row table, and creating a covering index eliminated it entirely.
4. Explain the Lenovo Channel Data Management System.
   - **Answer:** CDMS is a data consolidation platform for Lenovo India's channel partner ecosystem. It aggregates sales, inventory, and incentive data from hundreds of partners into a single MSSQL-based warehouse, then runs complex ETL pipelines to generate reconciled reports. My primary contribution was taking ownership of the most expensive stored procedures — the ones that ran for hours and blocked the nightly reporting window — and systematically optimizing them by analyzing execution plans, restructuring joins, and adding targeted indexes until the entire batch completed in under 12 minutes.
   - **If asked more:** The four ETL stages are staging, validation, transformation, and reporting. The worst bottleneck was in the transformation stage where a cursor-based loop processed partners one-by-one instead of as a set.
5. What problem did CDMS solve for Lenovo India?
   - **Answer:** Before CDMS, Lenovo India's channel team relied on manual Excel-based reconciliation with 200+ partners, which took days and was error-prone. CDMS automated the entire data pipeline — ingestion from partner systems, validation against Lenovo's records, incentive calculation, and report generation — so that the business had accurate, audit-ready numbers every morning instead of waiting a week. My performance optimization directly ensured that this automated pipeline could complete within the overnight batch window instead of spilling into business hours.
   - **If asked more:** The downstream reporting team started at 8 AM, so a 5-hour procedure meant stale data in executive dashboards and delayed decision-making on partner incentives.
6. What were the main modules in CDMS?
   - **Answer:** CDMS had four core modules: the Data Ingestion module that pulled raw files from partner SFTP sites, the Validation Engine that checked data quality against Lenovo's master records, the Calculation Engine that ran stored procedures for incentives and discounts, and the Reporting module that generated PDF and Excel outputs for the channel team. I worked most heavily on the Calculation Engine — specifically the stored procedures that aggregated partner-level sales and computed tier-based rebates, which were the worst-performing part of the system.
   - **If asked more:** Raw partner data went through staging tables, then validation sprocs flagged mismatches, then clean data fed into the calculation sprocs I optimized, and finally the results were materialized into reporting views.
7. What were your exact responsibilities in CDMS?
   - **Answer:** I was responsible for optimizing the MSSQL stored procedures that drove the nightly incentive calculation batch — I profiled each one, captured actual execution plans, identified the heaviest operators, and rewrote queries or added indexes as needed. I also owned the correctness validation: after every change, I ran the old and new procedures side-by-side, compared row counts and totals at each stage, and only deployed when I could prove bit-for-bit identical results. Beyond optimization, I contributed to API development for the reporting dashboard and helped debug production issues when the batch failed mid-cycle.
   - **If asked more:** I used checksums and row-count comparison scripts that compared output tables before and after optimization, and I ran them against a production-sized test dataset, not just a subset.
8. How did you reduce stored procedure execution time from 5 hours to under 12 minutes?
   - **Answer:** I started by running the procedure with `SET STATISTICS TIME ON` and capturing the actual execution plan from SSMS — that immediately showed me a clustered index scan on a 50-million-row fact table caused by a join predicate that wasn't SARGable. I rewrote the join condition to be index-friendly, added two missing non-clustered indexes that the execution plan recommended, and replaced a cursor-based loop with a single set-based `UPDATE` statement. Each change was tested independently: I measured the improvement, verified correctness, and only then moved to the next bottleneck.
   - **If asked more:** The cursor processed one partner at a time across 200 partners, and replacing it with a single `UPDATE` with a `JOIN` eliminated 199 round trips and reduced the step from 45 minutes to 90 seconds.
9. What kind of execution plan issues did you identify?
   - **Answer:** The most common issue was table scans on large fact tables caused by implicit type conversions — a `VARCHAR` column being compared to an `NVARCHAR` parameter forced SQL Server to scan the entire clustered index instead of seeking. I also found nested loop joins when the outer table had millions of rows, which should have been hash matches; key lookups on non-covering indexes that added millions of unnecessary page reads; and a cursor-based loop that processed partners one-by-one instead of as a set.
   - **If asked more:** I diagnosed implicit conversions by looking for the yellow warning triangles in the execution plan and then checking the actual data types of the columns vs. the parameters in the `WHERE` clause.
10. What kind of indexes did you create or modify?
    - **Answer:** I created mostly non-clustered covering indexes that included all columns referenced in the `SELECT`, `WHERE`, and `JOIN` clauses so that SQL Server could satisfy the query entirely from the index without touching the table. For one critical procedure, I modified an existing composite index by reordering the key columns — the original had `Status` first and `Date` second, but the query filtered on `Date` ranges and only then on `Status`, so swapping the order turned a scan into a seek. I also added filtered indexes for queries that always targeted a specific status value, which kept the index small and fast.
    - **If asked more:** I used the `INCLUDE` clause to add payload columns without bloating the index key, and verified improvement by checking that key lookups disappeared from the execution plan.
11. How did you validate that optimized stored procedures still returned correct results?
    - **Answer:** I built a validation harness in T-SQL that ran the old and new versions of the procedure side-by-side against the same production-mirror database. For every output table, I compared row counts, checksums, and min/max/sum of key numeric columns, and I flagged any discrepancy down to the individual row level. I also tested edge cases — empty input sets, duplicate records, NULL values in join columns — to make sure the optimizations didn't silently break business logic.
    - **If asked more:** I used `CHECKSUM_AGG(BINARY_CHECKSUM(*))` per table to get a fingerprint of the entire result set and compared fingerprints between old and new runs.
12. How did you test performance under concurrent load?
    - **Answer:** I wrote a PowerShell script that launched 10 parallel `sqlcmd` sessions to mimic the real-world scenario where multiple reports kicked off at the same time, and I monitored `sys.dm_exec_requests` to see blocking and wait statistics. This exposed a blocking issue where the optimized procedure held a table-level lock — I fixed it by adding index-level locking hints and breaking a long-running transaction into smaller batches.
    - **If asked more:** I used `WITH (ROWLOCK)` on the target table to allow concurrent reads, and tested that it didn't degrade performance under single-user load.
13. What was the most difficult part of optimizing CDMS?
    - **Answer:** The hardest part was untangling a 400-line stored procedure that had been modified by four different developers over two years — it had nested transactions, GOTO statements, and temp tables that were created and dropped in confusing order. I couldn't just rewrite it from scratch because I needed to preserve every business rule. I ended up refactoring it step by step: I extracted the cursor logic into a set-based CTE, consolidated the temp tables into a single table variable, and tested each block independently before merging. The overall time dropped from 5 hours to 12 minutes, but the refactoring alone took two weeks of careful work.
    - **If asked more:** A GOTO-based error handler was masking a real bug — it caught an error, logged it, then jumped back into the middle of the loop, causing duplicate processing of certain partners.
14. What is partner tagging in CDMS?
    - **Answer:** Partner tagging is a classification system in CDMS where each channel partner is assigned metadata tags — like region, tier, product specialization, or incentive program — that drive how their data is processed and reported. For example, a "Platinum-tier" partner in the "North" region might get different rebate calculations and a different report format than a "Silver-tier" partner in "South." My optimization work directly impacted partner tagging because the stored procedures I optimized used these tags as join and filter criteria, and the original queries had poor index usage on the tag lookup tables.
    - **If asked more:** Tags were stored in a normalized `PartnerTag` junction table with `PartnerID` and `TagID`, and a missing composite index on `(TagID, PartnerID)` caused a scan for every tag-based filter.
15. How did input validation improve reporting accuracy?
    - **Answer:** Before my validation work, raw partner data was loaded directly into the calculation engine — if a partner sent a duplicate serial number or a date in the wrong format, the procedure would silently produce incorrect incentive totals. I added a validation layer in MSSQL that ran before the main calculation: it checked data types, looked for duplicates against the baseline inventory, flagged records that referenced non-existent partners, and quarantined failures into an exception table. This stopped bad data from polluting the reports and gave the operations team a clear list of what to fix, which is what drove the 90% reduction in reconciliation effort.
    - **If asked more:** For duplicate serial numbers, I used `ROW_NUMBER() OVER (PARTITION BY SerialNumber ORDER BY CreatedDate)` and only kept `RowNumber = 1` for processing.
16. What were the 4 ETL pipeline stages in CDMS?
    - **Answer:** Stage 1 was **Extract** — raw files from partner SFTP servers were pulled and staged into flat staging tables with minimal transformation. Stage 2 was **Validate** — the validation engine checked data completeness, format correctness, and business rule compliance, flagging failures into exception queues. Stage 3 was **Transform** — the clean data was aggregated, joined with master data, and run through the calculation stored procedures I optimized to compute incentives and rebates. Stage 4 was **Load** — the final results were written to reporting tables and pushed to the partner portal and executive dashboards.
    - **If asked more:** Stage 3 (Transform) was the bottleneck — specifically the incentive calculation procedure that took 5 hours because of the cursor-based partner loop and missing indexes.
17. How did you reduce reporting discrepancies by 90%?
    - **Answer:** We achieved that by shifting from manual partner-sent spreadsheets to an automated, validated pipeline where discrepancies were caught at ingestion time rather than at reconciliation time. Before CDMS, the operations team would export partner data, manually compare it against Lenovo's records in Excel, and chase partners for corrections — that took days and was error-prone. After, if a partner sent a serial number that was already claimed, the system flagged it instantly, held it in an exception table, and the report only included verified records.
    - **If asked more:** We tracked the count of post-report manual correction tickets raised by the finance team before and after CDMS, and the quarterly average dropped from 40+ tickets to under 4.
18. What does 60% throughput improvement mean in your project?
    - **Answer:** In the cold-chain project, 60% throughput improvement means the system could handle 60% more sensor messages per second from the IoT gateways without any increase in infrastructure cost. Before optimization, our Kafka consumers processed about 500 messages per second before lag started building up; after we tuned the consumer batch sizes, adjusted the Spring Boot thread pool, and added Redis caching for duplicate detection, the same consumers handled 800 messages per second with stable lag.
    - **If asked more:** The duplicate-detection query was hitting MSSQL on every message, adding 15ms latency per message; moving the dedup check to Redis with a TTL-based set cut that to under 1ms and eliminated the bottleneck.
19. Explain the Real-Time Partner Inventory Management System.
    - **Answer:** This system was built to solve a trust problem between Lenovo and its 200+ channel partners — each partner reported their inventory independently, and the numbers never matched Lenovo's baseline. I built a centralized MSSQL validation engine that took Lenovo's verified baseline data as the single source of truth and ran every incoming partner inventory record against it. The engine validated serial numbers, flagged duplicates, rejected records that didn't match the baseline, and calculated the true inventory in under 15 seconds per partner — all in set-based T-SQL so it scaled linearly.
    - **If asked more:** For each partner batch, it joined the incoming serial list against the baseline using `LEFT JOIN` and `NOT EXISTS` to identify valid, duplicate, and invalid records in a single pass.
20. What problem did the inventory validation engine solve?
    - **Answer:** The problem was that Lenovo's channel team had no automated way to reconcile inventory across 200+ partners — each partner sent their stock data in different formats, often with duplicates or serials that didn't match Lenovo's distribution records. The manual reconciliation process took a team of five people nearly a week every month, and even then, discrepancies of 10-15% were common. The validation engine automated the entire process: it ingested partner data, validated every serial number against the trusted baseline, and produced a reconciled inventory count in seconds, cutting reconciliation effort by 90%.
    - **If asked more:** One large partner was double-reporting the same 500 serial numbers across two different product lines, inflating their inventory significantly until the engine caught it.
21. How did you calculate inventory from verified baseline data?
    - **Answer:** The baseline data was a table of serial numbers that Lenovo had shipped to each partner, with status columns like `Shipped`, `In-Transit`, `Delivered`, and `Sold`. When a partner submitted their current inventory, I joined their serial list against the baseline using MSSQL set operations: records in both lists were marked as verified, records in the baseline but missing from the partner list were flagged as potentially sold or lost, and records in the partner list but not in the baseline were flagged as invalid. The final inventory count was simply the count of verified `Delivered` serials minus any serials the partner had reported as sold in previous cycles.
    - **If asked more:** I used `LAG()` to compare the current submission against the partner's previous submission and automatically detected serials that moved from "in stock" to "missing" without a corresponding sale report.
22. What were the common inventory data issues?
    - **Answer:** The most common issue was **duplicate serials** — the same serial number appearing in two different partner submissions, either because partners accidentally double-submitted or because serials were being passed between partners in the gray market. The second issue was **invalid serials** — partners reporting serials that Lenovo had never shipped to them, often due to data entry errors or mixing up stock from different distributors. Third was **missing serials** — partners reporting fewer units than Lenovo's baseline showed, which could mean unreported sales or theft.
    - **If asked more:** If Serial X was shipped to Partner A but reported by Partner B, the system flagged it as a cross-partner exception and escalated to the channel sales manager.
23. How did you handle duplicate inventory records?
    - **Answer:** I built a multi-layered deduplication strategy. First, within a single partner submission, I used `ROW_NUMBER() PARTITION BY SerialNumber ORDER BY ReportedDate DESC` to keep only the most recent occurrence of each serial. Second, across submissions, I maintained a history table of all previously verified serials — if a serial appeared again in a new submission from a different partner, it was flagged as a cross-partner duplicate and quarantined for manual review. The dedup logic was entirely in MSSQL and ran as part of the validation stored procedure.
    - **If asked more:** The cross-partner dedup was a simple `EXISTS` check against the verified inventory history table indexed on `SerialNumber` with a filtered index for active records only.
24. How did you handle invalid serial records?
    - **Answer:** Invalid serials were records submitted by a partner that didn't exist in Lenovo's baseline distribution table at all. My validation procedure used a `LEFT JOIN` where the baseline table was on the right side, and if the join produced a NULL baseline serial, the record was flagged as invalid. These invalid records were written to an exception table with the partner ID, serial number, and a reason code, and the main inventory calculation excluded them entirely. The partner portal displayed these exceptions in real time so the partner could correct their submission and resubmit.
    - **If asked more:** The exception flow was: invalid record → exception table → email notification to partner → partner resubmits corrected data → validation re-runs → record moves from exception to verified.
25. How did you process 10,000+ serial records?
    - **Answer:** I processed them using set-based T-SQL operations — I loaded all 10,000+ serials into a table-valued parameter in a single batch, then ran validation, dedup, and baseline joins against that set in one pass per partner. The key was avoiding RBAR (row-by-agonizing-row) processing: instead of looping through each serial number, I used `MERGE` statements and set-based `UPDATE` with `JOIN` to validate the entire batch in milliseconds. Even with 200 partners and 10,000+ serials each, the total processing time stayed under 15 seconds because every operation was index-optimized and set-based.
    - **If asked more:** I passed the entire partner submission as a structured TVP with columns `SerialNumber`, `ProductCode`, `ReportedDate`, and MSSQL processed it as a single relational set.
26. How did you reduce inventory processing time to under 15 seconds?
    - **Answer:** The original approach processed each serial number individually through a cursor, which cost about 50ms per serial — for a partner with 10,000 serials, that alone was 500 seconds. I replaced the cursor with a single `INSERT INTO #validated SELECT s.* FROM @submitted s JOIN Baseline b ON s.SerialNumber = b.SerialNumber` — a set-based operation that validated all 10,000 serials in under 100ms. Then I applied the same approach to deduplication and exception handling: every loop was replaced with a set-based join or window function, and I added covering indexes on the `SerialNumber` column in all temp and base tables.
    - **If asked more:** The cursor version had 10,000 individual index seeks (one per serial), while the set-based version had a single hash match join with 10,000 input rows.
27. How did you measure 94% inventory accuracy?
    - **Answer:** We measured accuracy by comparing the system's validated inventory count against physical stock audits conducted by Lenovo's channel team. After processing all 200+ partner submissions through the validation engine, we ran spot-check physical audits for a random sample of 20 partners each quarter. The 94% figure means that for those audited partners, the system's inventory count matched the physical count within a 1% tolerance 94% of the time — up from approximately 72% before the validation engine was implemented.
    - **If asked more:** We used stratified random sampling by partner tier (Platinum, Gold, Silver) with 95% confidence interval and calculated the weighted average accuracy across all tiers.
28. How did you reduce reconciliation effort by 90%?
    - **Answer:** Before the engine, reconciliation meant five people spending 3-4 days manually comparing Excel sheets from 200+ partners against Lenovo's SAP exports. After the engine, the entire process was automated: partners submitted data through a portal, the system validated and reconciled in 15 seconds, and the exceptions were displayed in a dashboard. The operations team only had to handle the 6% of records that were flagged as exceptions, rather than manually checking 100% of submissions. That's where the 90% reduction in effort came from — they went from checking 200,000 serials manually to reviewing 12,000 exceptions.
    - **If asked more:** Before: 5 people × 4 days × 8 hours = 160 person-hours per cycle; after: 1 person × 4 hours = 4 person-hours per cycle; (160-4)/160 = 97.5% reduction.
29. What was the business impact of the inventory system?
    - **Answer:** The biggest business impact was that Lenovo's channel finance team could finally close the monthly inventory reconciliation in one day instead of one week — that meant partner incentives could be calculated and paid on time, which improved partner satisfaction and reduced disputes. The 94% accuracy rate also meant that Lenovo had reliable data for demand forecasting and production planning — they weren't overproducing because of inflated partner-reported inventory numbers.
    - **If asked more:** The reduction in overproduction due to accurate inventory data saved significant working capital annually, and the operations team was freed to focus on partner relationships instead of data entry.
30. Explain the Cold-Chain Monitoring System.
    - **Answer:** This was a real-time IoT platform that monitored temperature and humidity in cold-storage units across Lenovo's supply chain. The data flow started with IoT gateways in each cold room sending MQTT messages to AWS IoT Core, which triggered a Lambda function that published the messages to a Kafka topic running on EC2. Our Spring Boot microservices consumed from Kafka, applied validation and deduplication, stored the time-series data in InfluxDB, and served it through REST APIs to a React dashboard and a Grafana instance. I worked on the entire backend pipeline — the Kafka consumer configuration, the Spring Boot APIs, the InfluxDB schema design, and the Redis caching layer — and we ultimately reduced temperature excursions by 85% and energy costs by 18%.
    - **If asked more:** A sensor reading of 8°C (above the 6°C threshold) → IoT Core → Lambda → Kafka → Spring Boot → InfluxDB write → Grafana alert triggers → operations team gets notified within 30 seconds.
31. What was your role in the Cold-Chain Monitoring System?
    - **Answer:** I was responsible for designing and implementing the backend data pipeline — I configured the Kafka consumer group with 6 partitions and 3 consumers to handle sensor bursts, wrote the Spring Boot service that deserialized, validated, and deduplicated incoming messages before writing to InfluxDB, and set up the Redis cache to store the latest reading per sensor so the dashboard could show real-time data without hitting the database. I also integrated the Grafana alerting rules that detected temperature excursions and triggered email and Slack notifications, and I containerized the entire Spring Boot application using Docker and set up the Jenkins CI/CD pipeline for automated deployment.
    - **If asked more:** A key challenge was tuning the Kafka `max.poll.records` and `fetch.max.wait.ms` to balance latency vs. throughput when sensor data burst from 100 msg/s to 800 msg/s during peak seasons.
32. Why was IoT needed in the cold-chain project?
    - **Answer:** IoT was essential because cold-chain inventory is perishable and time-sensitive — a refrigerator unit failing at 2 AM could spoil products worth crores before anyone noticed without automated monitoring. Manual temperature logging (someone checking a thermometer twice a day) was unreliable and couldn't detect transient excursions that lasted only 15 minutes. IoT gave us continuous, real-time telemetry from every cold-storage unit, automated alerts the moment a threshold was breached, and historical data for root-cause analysis and energy optimization — all without human intervention.
    - **If asked more:** We placed 3-4 sensors per cold room (top, middle, bottom, near door) to detect hot spots and door-open events, with readings every 30 seconds.
33. How did data flow from gateways to dashboards?
    - **Answer:** The gateway devices in each cold room published JSON payloads over MQTT to AWS IoT Core, which had a rule that forwarded every message to a Lambda function. The Lambda did a lightweight parse and then produced the message to a Kafka topic on EC2. Our Spring Boot Kafka consumer picked up the message, validated it (checked for null fields, out-of-range values, duplicate message IDs), enriched it with metadata (room name, product type), and wrote it to InfluxDB. The Grafana dashboard queried InfluxDB directly for live charts, while the React dashboard called our Spring Boot REST APIs that aggregated data from both InfluxDB and Redis for the most recent readings.
    - **If asked more:** We didn't use Lambda straight to InfluxDB because Lambda has a 15-minute timeout and no built-in retry queue for downstream failures, while Kafka gave us persistent buffering, replay capability, and ordered processing.
34. Why did you use AWS IoT Core?
    - **Answer:** We used AWS IoT Core because it's a fully managed MQTT broker that handles device authentication, TLS termination, and topic-based routing out of the box — we didn't want to run and scale our own MQTT broker on EC2. It also integrates natively with Lambda via IoT rules, so we could process incoming messages without writing any polling code. The device shadow feature was useful for tracking the last known state of each gateway, and the IoT Core policies let us restrict each device to only publish to its own topic.
    - **If asked more:** Each gateway had an X.509 certificate installed at manufacturing time, and IoT Core validated the certificate on connect and applied the policy that scoped the device to its `/<location>/<room>/<sensor-id>` topic.
35. Why did you use AWS Lambda?
    - **Answer:** Lambda served as the lightweight bridge between AWS IoT Core and Kafka — it received the MQTT message from IoT Core via a rule, deserialized the JSON, added a timestamp if missing, and published it to Kafka. We chose Lambda over a persistent EC2 consumer because IoT message volume was spiky (near-zero at night, 800 msg/s during peak) and Lambda scales to zero when not needed. The function was simple — about 50 lines of Python — and its only job was to transform the MQTT payload into a Kafka-compatible format and handle any parsing errors gracefully.
    - **If asked more:** If Lambda failed to parse a message or the Kafka producer threw an exception, the function wrote the raw payload to an S3 bucket with a timestamp and error code for later replay.
36. Why did you use Kafka in the cold-chain project?
    - **Answer:** We needed a durable, scalable buffer between the IoT ingestion layer and the downstream analytics pipeline because the data rate was unpredictable — sensors published every 30 seconds per room, but with 200+ rooms the aggregate rate could spike to 800 messages per second. Kafka gave us the ability to absorb those spikes without back-pressuring the IoT Core → Lambda path, and it decoupled the producers (Lambda) from the consumers (Spring Boot) so that if the analytics pipeline went down for maintenance, no sensor data was lost.
    - **If asked more:** A single topic `cold-chain-sensor-data` with 6 partitions, messages partitioned by `room-id` for ordering, and 3 Spring Boot consumers in the same group for parallel processing.
37. Why was Kafka running on EC2?
    - **Answer:** We ran Kafka on EC2 instead of using MSK because the project was cost-sensitive — at the time, MSK's minimum cluster cost was significantly higher than running a single `m5.large` instance with Kafka and Zookeeper containerized in Docker. For our throughput requirements (800 msg/s peak, 500 GB retention for 7 days), a single well-tuned EC2 instance was sufficient. If the project scaled to 10x the volume, we would have migrated to MSK, but for initial deployment, EC2 was the pragmatic choice.
    - **If asked more:** We used gp3 EBS volumes with 3000 IOPS for Kafka logs, JVM heap set to 8 GB, `log.retention.hours=168`, and `unclean.leader.election.enable=false` to prevent data loss.
38. How did you design Kafka topics?
    - **Answer:** I designed a single topic called `cold-chain-sensor-data` with 6 partitions — one topic was enough because all sensor messages had the same schema and processing logic. The key for partitioning was the `room-id`, which guaranteed that all messages from the same cold room went to the same partition and thus were consumed in order. I also created a separate `cold-chain-alerts` topic with 3 partitions for alert events (excursions, device offline) so that the alerting consumers could scale independently of the sensor data consumers.
    - **If asked more:** We used `cleanup.policy=delete` for the main topic and `log.retention.ms=604800000` for 7-day retention to support historical replays.
39. How did you decide partition count?
    - **Answer:** I chose 6 partitions based on three factors: the target throughput of 800 msg/s, the consumer processing capacity of about 300 msg/s per Spring Boot instance, and the number of unique rooms (about 200). With 6 partitions and 3 consumers (2 partitions per consumer), each consumer handled roughly 270 msg/s, which was well under capacity and left headroom for spikes. The partition count also needed to be a multiple of the expected consumer count for balanced distribution.
    - **If asked more:** If we had used 100 partitions for 200 rooms, each consumer would handle 33 partitions, and a consumer failure would trigger a full rebalance of 100 partitions, taking 30+ seconds.
40. How did you handle sensor data bursts?
    - **Answer:** Sensor data bursts happened primarily during the morning and evening hours when cold-room doors were opened frequently for stock movement, causing the publish rate to spike from ~100 msg/s to ~800 msg/s. The architecture handled this naturally: AWS IoT Core accepted the burst and queued messages to Lambda, Lambda scaled up concurrency, and Kafka accepted the increased write rate because it had enough partition capacity. On the consumer side, I configured `max.poll.records=500` and `fetch.max.wait.ms=500` so that consumers fetched larger batches during bursts but still maintained low latency during normal periods.
    - **If asked more:** If Kafka consumer lag grew beyond 10,000 messages, a health check endpoint in Spring Boot reduced the consumer's `max.poll.records` dynamically to prevent memory exhaustion.
41. How did you handle delayed IoT messages?
    - **Answer:** Delayed messages occurred when a gateway lost network connectivity for a few minutes and then reconnected, sending all buffered messages at once with original timestamps. I handled this at the InfluxDB write layer: each data point was tagged with `ingestion_time` (when Spring Boot received it) and `sensor_time` (when the gateway recorded it). InfluxDB stored the data by `sensor_time`, so delayed messages were written to their correct time bucket even if they arrived late.
    - **If asked more:** Each gateway synced its clock via NTP every hour, and if the clock skew was greater than 5 seconds, the messages carried a `clock_error` flag stored as an InfluxDB tag for data-quality filtering.
42. How did you handle duplicate IoT messages?
    - **Answer:** Duplicate messages happened when a gateway retransmitted a message because it didn't receive an MQTT QoS 2 acknowledgment, or when Lambda's at-least-once delivery to Kafka produced duplicates. I implemented deduplication in the Spring Boot consumer using a combination of a unique `message_id` (UUID generated by the gateway) and a Redis SET with TTL. When a message arrived, the consumer checked if its `message_id` already existed in Redis: if yes, the message was skipped; if no, it was added to Redis with a 24-hour TTL and then written to InfluxDB.
    - **If asked more:** A primary-key check on MSSQL would have added 5-10ms latency per message; Redis with in-memory SET operations was orders of magnitude faster at 800 msg/s.
43. How did you handle invalid sensor readings?
    - **Answer:** I defined a set of validation rules in the Spring Boot consumer that ran before writing to InfluxDB: temperature had to be between -20°C and 60°C, humidity between 0% and 100%, and the `sensor_time` had to be within 5 minutes of the current time. If a message failed validation, it was written to a separate `invalid-readings` Kafka topic, and the Grafana dashboard had a panel showing the count of invalid readings per room over time to help identify failing sensors.
    - **If asked more:** A sensor once started reporting -40°C due to a hardware fault; the validation rule caught it, the invalid reading was logged, and the maintenance team replaced the sensor before it caused a false temperature excursion alert.
44. Why did you use InfluxDB?
    - **Answer:** We chose InfluxDB because it's purpose-built for time-series data — it handles millions of data points per second with automatic downsampling, retention policies, and continuous queries that would be impractical in a relational database. Each sensor reading (temperature, humidity, timestamp, room-id) was stored as a single point, and InfluxDB's columnar storage compressed the data by 10x. It also integrated natively with Grafana, so our dashboard queries were simple Flux queries without custom API middleware.
    - **If asked more:** Raw data was kept for 7 days at 30-second resolution, then downsampled to 5-minute averages for 30 days, and hourly averages for 1 year, using InfluxDB tasks.
45. Why did you use Grafana?
    - **Answer:** Grafana gave us production-ready dashboards with minimal effort — we connected it directly to InfluxDB and built real-time panels for temperature trends, humidity charts, and excursion alerts in a few hours. The alerting engine was critical: we configured Grafana to evaluate alert rules every 30 seconds, and when a temperature reading exceeded the threshold for more than 2 consecutive evaluations, it sent notifications via Slack and email.
    - **If asked more:** The main dashboard had a top row with real-time temperature gauges per cold room, a middle row with 24-hour trend lines with excursion zones highlighted in red, and a bottom row with alert history.
46. What kind of Grafana dashboards did you build?
    - **Answer:** I built three main dashboards. The **Operations Dashboard** showed a grid of all 200+ cold rooms, each with a color-coded tile (green=normal, yellow=warning, red=excursion), and clicking a tile opened a detailed 24-hour trend chart. The **Alerts Dashboard** listed all active and historical excursions with room name, threshold breached, duration, and resolution time. The **Energy Dashboard** displayed power consumption trends per cold room with a temperature overlay — this helped the team identify rooms that were overcooling and wasting energy.
    - **If asked more:** Operators could see that Room A consumed 30% more power than Room B for the same temperature range, leading them to discover a faulty door seal that was letting cold air escape, which drove the 18% cost reduction.
47. What are excursion alerts?
    - **Answer:** An excursion alert fires when a sensor reading stays outside an acceptable temperature or humidity range for longer than a configured duration. For example, if a vaccine cold room had a threshold of 2°C to 8°C, and the sensor reported 9°C for three consecutive readings (90 seconds), the system classified that as a minor excursion. If the temperature exceeded 10°C or stayed outside range for more than 5 minutes, it became a major excursion and triggered immediate Slack and SMS alerts to the on-call engineer.
    - **If asked more:** Severity levels were: minor (informational, auto-resolved), major (requires acknowledgment within 15 minutes), critical (requires immediate action, escalates to manager if unacknowledged after 5 minutes).
48. How did you reduce temperature excursions by 85%?
    - **Answer:** The 85% reduction came from shifting from reactive to proactive monitoring. Before the system, the operations team didn't know about an excursion until someone opened the cold room hours later. With real-time monitoring and alerts, the team could respond to a rising temperature trend within 30 seconds — they could check if a door was left open, if the compressor had failed, or if the setpoint had been changed. Additionally, historical data helped the team identify patterns, like solar heat gain through a window at 2 PM, and fix the root cause permanently.
    - **If asked more:** Measured as (excursions before system - excursions after system) / excursions before system × 100, comparing 6 months before deployment to 6 months after using the same set of cold rooms.
49. How did the system reduce energy costs by 18%?
    - **Answer:** The energy savings came from data-driven optimization of cooling setpoints. Historical InfluxDB data showed that several rooms were being overcooled — kept at 2°C when the product only required 6°C, which wasted significant energy. By analyzing temperature trends alongside power consumption, the operations team identified the optimal setpoint for each room based on its insulation quality, ambient temperature, and product requirements. Fixing issues like faulty door seals and insulation gaps directly reduced compressor runtime.
    - **If asked more:** The average cold room consumed about ₹15,000/month in electricity before; optimizing setpoints and fixing seals dropped it to ₹12,300/month, saving ₹2,700/month per room across 200 rooms.
50. How did you improve load time by 80%?
    - **Answer:** I improved the React dashboard's initial load time by implementing server-side pagination and lazy loading for the cold-room grid — originally, the API was returning all 200+ rooms with full sensor history, which was about 15 MB of JSON and took 12 seconds to parse and render. I changed the API to return only the latest reading per room with a summary status (green/yellow/red), which cut the payload to under 50 KB, and loaded room details on demand when the user clicked a tile. I also enabled Redis caching for the aggregate dashboard data with a 10-second TTL.
    - **If asked more:** API response time went from 3.2 seconds to 180ms, React render time from 8.5 seconds to 400ms, total perceived load time from ~12 seconds to ~2 seconds.
51. How did you improve responsiveness by 30%?
    - **Answer:** The 30% responsiveness improvement refers to the time between a sensor reading arriving at the backend and the dashboard reflecting that new value. The bottleneck was the polling interval: the React dashboard was polling the REST API every 15 seconds. I switched to WebSocket connections — the Spring Boot service pushed new readings to connected clients as soon as they were written to InfluxDB, which reduced update latency to under 500ms.
    - **If asked more:** I used Spring Boot's `WebSocketHandler` with a `SimpMessagingTemplate` that broadcast the latest reading to the `/topic/latest-readings` channel whenever a new data point was persisted.
52. What was the role of Redis in the cold-chain project?
    - **Answer:** Redis served three purposes. First, it was the **deduplication cache**: we stored `message_id` for each incoming message with a 24-hour TTL, so duplicate messages were detected in under 1ms. Second, it was the **latest-reading cache**: for each sensor, we stored the most recent valid reading so the dashboard's summary API could return the current state of all 200+ rooms instantly. Third, it was the **session store** for Spring Session — user sessions were stored in Redis so that any Spring Boot instance could serve any request without losing session state.
    - **If asked more:** The dedup cache stored ~200,000 keys at ~100 bytes each = ~20 MB; the latest-reading cache stored 600 keys at ~200 bytes each = ~120 KB — well within a single t3.micro instance's 500 MB allocation.
53. How did you secure APIs with Spring Security?
    - **Answer:** I configured Spring Security with a JWT-based authentication filter that intercepted all API requests except the public health-check endpoint. The filter extracted the `Authorization: Bearer <token>` header, validated the token's signature using an RSA public key, checked the token's expiration and issuer claims, and then set the `SecurityContext` with the user's roles and permissions. I also applied method-level security using `@PreAuthorize` annotations — admin endpoints were restricted to `ROLE_ADMIN`, dashboard data to `ROLE_VIEWER`. For the IoT ingestion API, I used API keys instead of JWT because sensors couldn't manage token refresh.
    - **If asked more:** The React app received an access token with 15-minute expiry and a refresh token with 7-day expiry; the frontend used an Axios interceptor to automatically refresh the access token when it expired.
54. How did JWT authentication work in your project?
    - **Answer:** When a user logged in through the React login page, the Spring Boot authentication endpoint validated the username/password against the MSSQL user table and returned a signed JWT access token (valid for 15 minutes) and a refresh token (valid for 7 days). The JWT contained the user's ID, roles, and a session ID in its claims, signed with an RSA private key so any service could verify the token using the public key. The React app stored the token in `localStorage` and sent it in the `Authorization` header for every API call.
    - **If asked more:** We used RSA instead of HMAC because with HMAC every service needs to share the same secret key; with RSA only the authentication service holds the private key.
55. Where did OAuth2 fit in your project?
    - **Answer:** We used OAuth2 for Grafana authentication integration — instead of managing separate Grafana user accounts, we configured Grafana to use OAuth2 with our Spring Boot application as the authorization server. When a user logged into Grafana, they were redirected to our login page, authenticated with their existing credentials, and Grafana received an access token for API calls.
    - **If asked more:** The `oauth2` section in `grafana.ini` had `auth_url`, `token_url`, and `api_url` pointing to our Spring Boot endpoints, with role mapping from JWT claims.
56. How did you use Swagger/OpenAPI?
    - **Answer:** I used SpringDoc OpenAPI to auto-generate Swagger documentation for all our REST APIs. The documentation was available at `/swagger-ui.html` and `/v3/api-docs` for each microservice. The Swagger UI became the primary reference for the frontend team — they could see every endpoint, its schema, authentication requirements, and test calls directly from the browser. The frontend team used the OpenAPI spec to generate TypeScript API clients using `openapi-generator`, which eliminated manual type definitions.
    - **If asked more:** We used `@Tag` annotations to group endpoints by domain (Sensor Data, Alerts, Configuration, Users), each with its own section in Swagger UI.
57. How did Jenkins reduce deployment time by 70%?
    - **Answer:** Before Jenkins, deployments were manual: build the JAR locally, SCP it to EC2, SSH in, stop the service, replace the JAR, restart, verify — about 30 minutes per deployment. I set up a Jenkins pipeline that automated the entire process: on every push to `main`, Jenkins checked out code, ran tests, built the Docker image, pushed it to Docker Hub, SSH'd into EC2, pulled the new image, stopped the old container, started the new one, and ran a smoke test — all in under 10 minutes. The pipeline also included automatic rollback if the smoke test failed.
    - **If asked more:** The Jenkinsfile had stages: Checkout → Test → Build → Dockerize → Deploy → Smoke Test → (on failure) Rollback, with post-build Slack notifications.
58. How did Docker help in deployment?
    - **Answer:** Docker eliminated the "it works on my machine" problem by packaging the Spring Boot application, its dependencies, and the JVM version into a single container image that ran identically on the developer's laptop, the Jenkins build server, and the production EC2 instance. It also made deployments atomic and reversible — we tagged each image with the Git commit hash, and rolling back was as simple as running with the previous image tag. We ran four containers on a single EC2 instance: Spring Boot API, Redis, Grafana, and Nginx — all managed with Docker Compose.
    - **If asked more:** We used a custom bridge network `cold-chain-net` so containers could communicate by service name (e.g., `api:8080`, `redis:6379`) instead of environment-specific IP addresses.
59. What was the role of AWS S3?
    - **Answer:** In the cold-chain project, S3 served as the long-term archival store for raw sensor data. While InfluxDB kept high-resolution data for 7 days and downsampled data for up to a year, all raw sensor messages were archived to S3 in Parquet format using a nightly batch job. This served two purposes: compliance (we needed to retain raw data for 3 years) and reprocessing (if we discovered a bug in our data pipeline, we could replay archived data). Each object was keyed by `year/month/day/room-id/hour.parquet` for efficient range queries.
    - **If asked more:** A Spring Boot scheduled task ran at midnight, queried the last 24 hours of raw data from a tracking table in MSSQL, serialized it to Parquet, and uploaded to S3 with server-side encryption.
60. What was the role of PostgreSQL?
    - **Answer:** We used PostgreSQL as the metadata and configuration database for the cold-chain project — it stored user accounts, role assignments, cold-room metadata, alert configuration rules, and audit logs. We chose PostgreSQL over MSSQL for this non-time-series workload because the operational overhead was lower and the `JSONB` column type was convenient for storing flexible alert rule configurations. The Spring Boot application used JPA/Hibernate to interact with PostgreSQL, and InfluxDB queries referenced room IDs that were joined against PostgreSQL metadata in Grafana.
    - **If asked more:** The `alert_rules` table had columns `room_id`, `metric` (temperature/humidity), `operator` (gt/lt), `threshold`, `duration_seconds`, `severity`, and a `channels` JSONB column for notification targets.
61. What was the role of MQTT?
    - **Answer:** MQTT was the communication protocol between the IoT gateways and AWS IoT Core. We chose MQTT over HTTP because it's lightweight (the binary header is only 2 bytes), supports persistent connections with minimal overhead, and has built-in QoS levels — we used QoS 1 (at-least-once delivery) for reliable delivery. Each gateway published messages to topics like `cold-chain/{location-id}/{room-id}/{sensor-id}`, and IoT Core used topic filtering to route each location's data to the appropriate Lambda function.
    - **If asked more:** We had 600 unique topics across 20 locations, 10 rooms per location, and 3 sensors per room; IoT Core wildcard subscription `cold-chain/+/+/+` captured all messages.
62. What was the role of Databricks?
    - **Answer:** Databricks was used for offline analytics and reporting on aggregated cold-chain data. While Grafana handled real-time operational dashboards, the business team needed weekly and monthly reports on energy consumption trends, excursion patterns by location, and sensor reliability statistics. We exported downsampled InfluxDB data to S3 as Parquet files daily, and Databricks notebooks read that data to generate trend analyses and predictive models.
    - **If asked more:** One use case was a linear regression on energy consumption vs. ambient temperature for each room, identifying rooms where the slope was abnormally high (indicating poor insulation) and generating a prioritized maintenance work order list.
63. What was the role of DLT?
    - **Answer:** DLT (Delta Live Tables) in Databricks was used to build and maintain the ETL pipeline that transformed raw sensor archives into clean, analytics-ready tables. Instead of writing manual Spark jobs with fragile scheduling, DLT let us define the pipeline declaratively — we specified the source (S3 Parquet files), the transformations (deduplication, filtering invalid readings, joining with room metadata), and the target Delta tables, and DLT handled incremental processing, data quality checks, and automatic retries.
    - **If asked more:** The "bronze" table ingested raw Parquet from S3, the "silver" table applied deduplication and validation, and the "gold" table joined with room metadata for the business reporting layer.
64. What was the most challenging project you worked on?
    - **Answer:** The CDMS stored procedure optimization was the most challenging because it wasn't just a technical problem — I had to understand the business meaning of every column, every join, and every temp table in a 400-line procedure that nobody had fully documented. The pressure was high because the nightly batch had been failing for weeks, and the business team needed their reports. I had to balance speed with safety — I couldn't afford to introduce a calculation error that would affect partner incentives. The approach of making one change at a time, testing it against the full dataset, and only then moving to the next bottleneck was what made it work.
    - **If asked more:** After three days of analysis, I found that a `WHERE CAST(date_column AS DATE) = '2024-01-01'` was forcing a full table scan because the `CAST` made the predicate non-SARGable; rewriting it to a range condition alone cut 45 minutes from the runtime.
65. Which project had the highest business impact?
    - **Answer:** The inventory validation engine had the highest direct business impact because it saved Lenovo's channel finance team an entire week of manual work every month and gave them trustworthy inventory numbers for the first time. Before the system, the finance team couldn't close the monthly books on time because inventory discrepancies took days to resolve. After the system, reconciliation was a 15-second automated process with a clear audit trail, and the finance team could close the month on the first working day.
    - **If asked more:** One quarter before the system, a partner dispute over 2,000 serial numbers took 3 weeks of email exchanges and physical audits to resolve; after the system, the same scenario was resolved in 15 seconds.
66. Which project had the most technical complexity?
    - **Answer:** The cold-chain IoT monitoring system was the most technically complex because it involved the widest range of technologies and failure modes — hardware failures (sensors dying, gateways disconnecting), network issues (MQTT disconnections, Kafka broker restarts), data quality problems (duplicates, delayed messages, corrupt payloads), and real-time performance requirements all at once. It wasn't enough to make each component work individually — they all had to work together reliably, and a failure in any layer could cascade.
    - **If asked more:** One complex failure was a Kafka broker disk filling up because retention wasn't configured correctly after a schema change increased message size by 3x; consumers couldn't commit offsets, which caused rebalancing and duplicate downstream writes.
67. What would you improve if you redesigned one of your projects?
    - **Answer:** If I redesigned the cold-chain project, I would use Kafka with Tiered Storage or Confluent Cloud instead of self-managed Kafka on EC2. Managing Kafka ourselves meant handling broker restarts, partition rebalancing, disk space monitoring, and OS patching — all of which added operational overhead. I would also add a schema registry from day one to prevent silent data loss from message format changes, and implement end-to-end tracing with OpenTelemetry for faster debugging.
    - **If asked more:** With Avro and Schema Registry, the producer would have to register the new schema and the consumer would automatically reject incompatible changes, preventing silent data loss.
68. What production issue did you face and how did you solve it?
    - **Answer:** One critical production issue was that the Kafka consumer lag started growing unboundedly during peak hours, reaching 500,000 messages after 4 hours, which meant the dashboard was showing data 4 hours old. I initially suspected the consumer was too slow, but after analyzing logs, I found that InfluxDB was throwing `partial write` errors because we had exhausted write batch capacity. The root cause was that the Spring Boot consumer's `@KafkaListener` was configured with `concurrency=1` — all 6 partitions were assigned to a single thread. I fixed it by increasing `concurrency=3`, tuning `max.poll.records` from 500 to 200, and adding retry with exponential backoff for InfluxDB writes.
    - **If asked more:** We had a Grafana panel showing Kafka consumer lag per partition, and a PagerDuty alert fired when lag exceeded 10,000 messages for more than 5 minutes.
69. How do you explain your project to a non-technical person?
    - **Answer:** For the cold-chain project: "Imagine you have 200 refrigerators spread across India, each storing products that spoil if the temperature goes wrong for more than a few minutes. Before our system, someone had to walk to each refrigerator and check a thermometer twice a day — which meant a broken refrigerator could go unnoticed for 12 hours and spoil everything inside. We installed sensors that send temperature data every 30 seconds, and our system automatically alerts the maintenance team the moment something starts going wrong — often before the product is even affected. This cut spoiled products by 85% and saved 18% on electricity bills."
    - **If asked more:** I follow the "before and after" framework — describe the pain before (manual checking, spoilage, late discovery) and the improvement after (automated, immediate, data-driven).
70. How do you explain your project architecture to a senior engineer?
    - **Answer:** I would describe the cold-chain architecture as a six-stage event-driven pipeline: IoT gateways publish MQTT to AWS IoT Core (QoS 1), which triggers a Python Lambda that produces to a 6-partition Kafka topic on EC2. A Spring Boot Kafka consumer group (3 instances, each handling 2 partitions) validates, deduplicates using Redis SETs with 24-hour TTL, writes time-series data to InfluxDB (with 7-day retention and downsampling), and caches latest readings in Redis. Grafana queries InfluxDB for real-time dashboards and alerts, while a React dashboard uses WebSocket for live updates.
    - **If asked more:** The critical design decisions were: Kafka for decoupling (handles bursts, provides replay), Redis for sub-millisecond dedup and caching (avoids InfluxDB load), and InfluxDB's automatic downsampling for cost-effective long-term storage.

---

## Java Core Questions

1. What are the main features of Java?
   - **Answer:** Java is platform-independent, object-oriented, has automatic memory management, strong typing, and built-in multithreading. In my projects, I rely on OOP for modular service/controller layers and on garbage collection so I rarely worry about manual memory management in Spring Boot apps.
   - **If asked more:** I can dive into platform independence via bytecode and JVM, explain how the classloader works, and contrast Java's memory model with languages like C++. I would also mention how features like try-with-resources helped me write cleaner DB resource handling.
2. What is platform independence in Java?
   - **Answer:** Java achieves platform independence through bytecode that runs on the JVM. I deploy the same Spring Boot JAR on Windows dev machines and Linux EC2 servers without recompiling, which is critical for our CI/CD pipeline.
   - **If asked more:** I can explain the compilation process (.java to .class), how JVMs exist per platform, and discuss portability challenges like file path separators or character encodings I encountered in cross-platform deployments.
3. What is JVM, JRE, and JDK?
   - **Answer:** JVM executes bytecode, JRE includes JVM plus core libraries, and JDK adds development tools like javac and javap. In my daily work, I only install JDK on dev machines and use JRE-based Docker images for production.
   - **If asked more:** I can explain the JVM architecture (class loader, runtime data areas, execution engine), how JIT compilation works, and why choosing the right JVM implementation matters for server performance.
4. What is bytecode?
   - **Answer:** Bytecode is the intermediate representation of Java source code, stored in .class files and executed by the JVM. I never work with bytecode directly, but understanding it helped me debug classpath issues in my CDMS project when compiled classes were stale.
   - **If asked more:** I can explain how javac produces bytecode, how the JVM verifies it before execution, and how tools like javap can inspect bytecode for troubleshooting generic type erasure or lambda desugaring.
5. What is the difference between stack and heap memory?
   - **Answer:** Stack stores primitive values and object references per thread, while heap stores all actual objects and is shared across threads. In my inventory project, large collections of serial records lived on heap while local loop variables stayed on stack.
   - **If asked more:** I can explain stack frames, how recursion causes StackOverflowError, how heap is divided into young/old generations, and how I tuned JVM heap settings to avoid OutOfMemoryError during bulk data processing.
6. What is a class?
   - **Answer:** A class is a blueprint for creating objects in Java. In my backend code, every entity like `Partner`, `SensorReading`, or `InventoryRecord` is a class with fields and methods.
   - **If asked more:** I can explain class members (fields, methods, constructors), static vs instance context, how a class is loaded into memory, and how I structure classes using layered architecture in Spring Boot.
7. What is an object?
   - **Answer:** An object is a runtime instance of a class with its own state. When I call `partnerRepository.findById(id)`, the returned `Partner` object has specific field values representing a real database row.
   - **If asked more:** I can explain object creation with `new`, memory allocation on heap, how the constructor initializes state, and object lifecycle from creation to garbage collection.
8. What are constructors?
   - **Answer:** Constructors initialize object state when an instance is created. In my DTOs and entities, I use constructors to set required fields so the object is never in an invalid state.
   - **If asked more:** I can explain default constructors, parameterized constructors, constructor chaining with `this()`, and why I prefer constructor injection over field injection in Spring services.
9. What is constructor overloading?
   - **Answer:** Constructor overloading lets me define multiple constructors with different parameters for flexible object creation. In my cold-chain project, I overloaded `Alert` constructors to accept either sensor ID alone or full sensor reading data.
   - **If asked more:** I can explain how Java differentiates overloaded constructors, why the `this()` call must be the first statement, and how it differs from builder patterns for complex object creation.
10. What is method overloading?
   - **Answer:** Method overloading means multiple methods share the same name but differ in parameters. I use it in my service layer when I need lookup methods with different filter criteria, like `findByPartnerId()` and `findByPartnerIdAndDate()`.
   - **If asked more:** I can explain compile-time polymorphism, how method resolution works, why return type alone cannot distinguish overloaded methods, and how autoboxing complicates overload resolution.
11. What is method overriding?
   - **Answer:** Method overriding allows a subclass to provide a specific implementation of a parent class method. In my projects, I override `equals()` and `hashCode()` in entity classes to ensure correct behavior in collections and JPA identity management.
   - **If asked more:** I can explain runtime polymorphism, the `@Override` annotation, covariant return types, the rule that overridden methods cannot be more restrictive, and how Spring AOP uses proxy-based overriding.
12. What is inheritance?
   - **Answer:** Inheritance lets a class acquire fields and methods from a parent class. In my CDMS project, I extended a base `AbstractReportService` to share common reporting logic across multiple report types.
   - **If asked more:** I can explain single inheritance in Java, the `extends` keyword, method overriding via inheritance, diamond problem with interfaces, and why I prefer composition over inheritance for maintainability.
13. What is encapsulation?
   - **Answer:** Encapsulation bundles data and methods together while hiding internal state via access modifiers. In my Spring Boot services, I keep entity fields private and expose behavior through public methods, preventing direct field manipulation.
   - **If asked more:** I can explain getters/setters, the principle of information hiding, how encapsulation supports maintainability, and why exposing internal collections directly can break encapsulation.
14. What is abstraction?
   - **Answer:** Abstraction hides complex implementation details and shows only essential features. I use abstraction when defining service interfaces in Spring Boot so controllers depend on contracts, not concrete implementations.
   - **If asked more:** I can explain abstract classes vs interfaces, how abstraction reduces coupling, how Spring's dependency injection supports programming to interfaces, and real examples from my layered architecture.
15. What is polymorphism?
   - **Answer:** Polymorphism allows objects to take multiple forms, behaving differently based on their actual type. In my cold-chain project, a single `NotificationSender` interface had `EmailSender` and `SmsSender` implementations triggered based on alert severity.
   - **If asked more:** I can explain compile-time (method overloading) vs runtime (method overriding) polymorphism, how the JVM uses vtable dispatch, and how Spring leverages polymorphism for dependency injection.
16. What is the difference between compile-time and runtime polymorphism?
   - **Answer:** Compile-time polymorphism is resolved during compilation through method overloading, while runtime polymorphism is resolved at runtime through method overriding. I use overloading for convenience methods and overriding for interface implementations in my projects.
   - **If asked more:** I can explain how javac resolves overloaded methods, how the JVM uses dynamic dispatch for overridden methods, performance implications, and how both forms appear together in Spring Boot code.
17. What is an interface?
   - **Answer:** An interface is a contract that defines method signatures without implementation. In my projects, I define repository and service interfaces to decouple layers and enable Spring to inject appropriate implementations.
   - **If asked more:** I can explain default and static methods in interfaces, functional interfaces vs marker interfaces, how interfaces support multiple inheritance of type, and when I choose an interface over an abstract class.
18. What is an abstract class?
   - **Answer:** An abstract class cannot be instantiated and may contain both abstract and concrete methods. In my CDMS project, I used an abstract `BaseValidationService` with common validation logic, leaving specific rules to subclasses.
   - **If asked more:** I can explain abstract method rules, constructors in abstract classes, access modifiers, how they differ from interfaces in terms of state and multiple inheritance, and when I choose abstract classes.
19. Difference between abstract class and interface.
   - **Answer:** Abstract classes can have state and constructors, while interfaces support only constants and abstract/default methods. I use abstract classes for shared logic across related classes and interfaces for defining contracts across unrelated classes.
   - **If asked more:** I can explain how Java 8 blurred the line with default methods, when to prefer one over the other, and examples from Spring like `JpaRepository` (interface) vs `AbstractPaginationHelper` (abstract class).
20. Can an interface have default methods?
   - **Answer:** Yes, interfaces can have default methods with a body introduced in Java 8. I used this when creating a custom functional interface for data transformation, providing a default no-op implementation so implementers only override what they need.
   - **If asked more:** I can explain the diamond problem with default methods, how to resolve conflicts with explicit override, why default methods were introduced for backward compatibility in Streams, and their limitations.
21. Can an interface have static methods?
   - **Answer:** Yes, interfaces can have static methods with a body since Java 8. I use static helper methods in interfaces for utility functions like `ValidationUtils.isValidSerial()` that belong conceptually to the validation domain.
   - **If asked more:** I can explain that interface static methods are not inherited, how they differ from class static methods, use cases like factory methods, and Java 9's private methods in interfaces.
22. What is the diamond problem?
   - **Answer:** The diamond problem occurs when a class inherits from multiple sources with conflicting default method implementations. Java avoids it with classes using single inheritance, and for interfaces, the implementing class must override the conflicting method.
   - **If asked more:** I can explain how Java 8's default methods reintroduced this risk, how explicit override resolves it, C++'s approach vs Java's, and real scenarios like extending multiple event listener interfaces.
23. What is the difference between `==` and `.equals()`?
   - **Answer:** `==` compares object references (memory addresses), while `.equals()` compares logical content. In my inventory validation, I used `.equals()` on serial numbers (Strings) to check logical equality, never `==`.
   - **If asked more:** I can explain how `equals()` default behavior mimics `==` for objects unless overridden, how String interning affects `==`, best practices for overriding `equals()`, and pitfalls with wrapper class comparisons.
24. What is the contract between `equals()` and `hashCode()`?
   - **Answer:** If two objects are equal via `equals()`, they must have the same hash code. I override both in my entity classes so that HashSets and HashMaps work correctly for deduplication and lookup operations.
   - **If asked more:** I can explain the full contract including the reverse (unequal objects CAN share hash codes), why using only one breaks hash-based collections, common implementations using Objects utility, and Lombok's `@EqualsAndHashCode`.
25. What happens if `hashCode()` is not overridden?
   - **Answer:** Without overriding `hashCode()`, the default Object implementation uses memory address. Two logically equal objects would have different hash codes, causing issues in HashSets and HashMaps like duplicates or lookup failures.
   - **If asked more:** I can explain how HashMap buckets work, consequences like memory leaks from duplicate entries, debugging such issues in production, and tools like IDE-generated hash code using prime numbers.
26. What is immutable class?
   - **Answer:** An immutable class cannot be modified after creation. In my cold-chain project, I used immutable DTOs for sensor readings to ensure thread-safe data transfer across Kafka consumer threads without synchronization.
   - **If asked more:** I can explain the rules (final class, private final fields, no setters, defensive copying in getters), why String is immutable, benefits like caching and thread safety, and when immutability hurts performance.
27. How do you create an immutable class?
   - **Answer:** I declare the class final, make all fields private final, initialize them through the constructor, provide only getters without setters, and return defensive copies for mutable fields. I used this approach for value objects in my inventory system.
   - **If asked more:** I can explain the builder pattern alternative for classes with many fields, how records simplify immutability in Java 14+, why Collections.unmodifiableList() is not full immutability, and serialization concerns.
28. Why is `String` immutable?
   - **Answer:** String immutability enables caching (String pool), security (no tampering of class names or DB URLs), thread safety, and efficient hash code caching. In my projects, I rely on String immutability when using Strings as HashMap keys.
   - **If asked more:** I can explain the String pool mechanism, how substring memory works pre-Java 7 vs now, why StringBuilder is needed for concatenation in loops, and how reflection could technically break immutability.
29. Difference between `String`, `StringBuilder`, and `StringBuffer`.
   - **Answer:** String is immutable, StringBuilder is mutable and not thread-safe, StringBuffer is mutable and thread-safe. I use String for fixed values, StringBuilder for single-threaded concatenation in loops, and rarely use StringBuffer in modern code.
   - **If asked more:** I can explain internal char[], capacity management, performance benchmarks, why StringBuilder is faster than StringBuffer, and how javac optimizes simple `+` concatenation using StringBuilder.
30. What is the String constant pool?
   - **Answer:** The String constant pool is a special heap region that caches String literals to save memory. When I write `String s = "partner"` in multiple places, all references point to the same pooled object.
   - **If asked more:** I can explain `intern()`, how literals vs `new String()` behave, pool location changes from permgen to heap, how String deduplication works in G1 GC, and memory implications for large datasets.
31. What is exception handling?
   - **Answer:** Exception handling uses try-catch-finally to manage runtime errors gracefully. In my Spring Boot APIs, I use global exception handlers with `@ControllerAdvice` to return consistent error responses instead of stack traces.
   - **If asked more:** I can explain the exception hierarchy (Throwable -> Exception/RuntimeException), checked vs unchecked, try-with-resources for closing DB connections, and custom exceptions I defined for business logic failures.
32. Difference between checked and unchecked exceptions.
   - **Answer:** Checked exceptions are checked at compile time and must be handled or declared; unchecked exceptions (RuntimeException) are not. I use checked exceptions for recoverable conditions like file not found, and unchecked for programming bugs like null pointer.
   - **If asked more:** I can explain when to use each type, best practices in Spring Boot (unchecked preferred for transactional rollback), how Spring wraps checked exceptions in DataAccessException, and custom exception design.
33. Difference between `throw` and `throws`.
   - **Answer:** `throw` actually throws an exception instance, while `throws` declares that a method might throw certain checked exceptions. In my code, I `throw` custom exceptions from service methods and declare `throws` in method signatures.
   - **If asked more:** I can explain exception propagation, how throws works with overriding, chained exceptions, and how Spring's declarative transaction management handles rollback for runtime exceptions.
34. Difference between `final`, `finally`, and `finalize`.
   - **Answer:** `final` is a keyword for constants, non-overridable methods, and non-inheritable classes. `finally` is a try-catch block that always executes. `finalize()` is a deprecated GC callback. In my projects, I use `final` for constants and `finally` for resource cleanup.
   - **If asked more:** I can explain how `finally` interacts with return statements, why `finalize()` should never be relied upon, alternatives like Cleaner and AutoCloseable, and real cleanup patterns in Spring Boot.
35. What is try-with-resources?
   - **Answer:** Try-with-resources automatically closes resources that implement AutoCloseable. I use it in my CDMS project for JDBC connections and file I/O, ensuring resources are closed even if an exception occurs, without needing a finally block.
   - **If asked more:** I can explain the multi-resource syntax, how resources are closed in reverse order, suppressed exceptions, how to make custom resources AutoCloseable, and Spring Boot's equivalent in JPA template methods.
36. What is a custom exception?
   - **Answer:** A custom exception is a user-defined class extending Exception or RuntimeException. In my inventory project, I created `InvalidSerialException` to handle specific validation failures distinctly from generic system errors.
   - **If asked more:** I can explain when to extend RuntimeException vs Exception, constructor best practices (message, cause), custom fields for error codes, integration with global exception handlers, and serialization UID.
37. What are access modifiers in Java?
   - **Answer:** Access modifiers control visibility: `private` (class only), `default` (package), `protected` (package + subclasses), `public` (everywhere). In my layered architecture, I keep fields `private` and decide method visibility based on which layer needs access.
   - **If asked more:** I can explain how access modifiers affect inheritance, package-private as the default, encapsulation benefits, how reflection bypasses access control, and module system (Java 9) restrictions.
38. What is static keyword?
   - **Answer:** `static` means a member belongs to the class, not instances. I use static constants for configuration keys, static utility methods for validation helpers, and static inner classes to group related types.
   - **If asked more:** I can explain static initialization blocks, why static methods cannot be overridden, how static variables are stored in the method area, thread safety concerns, and common anti-patterns like static service classes.
39. What is final keyword?
   - **Answer:** `final` on a variable makes it a constant, on a method prevents overriding, and on a class prevents inheritance. I mark service dependencies as `final` for immutability and use `final` constants for magic strings in configuration.
   - **If asked more:** I can explain blank final variables, why final parameters are useful in anonymous classes, how the JIT optimizes final methods, and how final helps with thread safety via safe publication.
40. What is transient keyword?
   - **Answer:** `transient` marks fields that should not be serialized. In my projects, I mark derived or cache fields as transient when entities are serialized for caching or cross-service communication.
   - **If asked more:** I can explain the serialization mechanism, how transient interacts with Externalizable, alternatives like `@JsonIgnore` in Jackson, and security concerns of serializing sensitive data.
41. What is volatile keyword?
   - **Answer:** `volatile` guarantees visibility of changes to a variable across threads, preventing thread-local caching. In my Kafka consumer configurations, I used volatile flags for graceful shutdown signals across threads.
   - **If asked more:** I can explain happens-before guarantees, why volatile does not provide atomicity (use AtomicInteger instead), common use cases (flags, double-checked locking), and comparison with synchronized.
42. What is serialization?
   - **Answer:** Serialization converts an object into a byte stream for storage or transmission. In my cold-chain project, sensor data objects were serialized when sent to Kafka topics and deserialized by consumer applications.
   - **If asked more:** I can explain Serializable interface, serialVersionUID importance, custom writeObject/readObject methods, how JSON serialization differs from Java serialization, and why Jackson is preferred in Spring Boot.
43. What is marker interface?
   - **Answer:** A marker interface has no methods but signals special behavior to the JVM or framework. `Serializable` and `Cloneable` are classic examples. In Spring Boot, `@Configuration` annotations serve a similar signaling purpose.
   - **If asked more:** I can explain how JVM checks for Serializable, why marker interfaces are considered a design pattern, the shift towards annotations as markers, and tradeoffs of custom marker interfaces vs annotations.
44. What is cloning?
   - **Answer:** Cloning creates a copy of an object using the `clone()` method of Object. In my inventory project, I cloned baseline configuration objects before applying partner-specific overrides to preserve the original.
   - **If asked more:** I can explain the Cloneable interface contract, why clone() is protected, shallow vs deep copy behavior, why copy constructors or factory methods are preferred over Cloneable, and serialization-based cloning.
45. What is shallow copy and deep copy?
   - **Answer:** Shallow copy copies only the top-level object, sharing references to nested objects. Deep copy recursively duplicates all referenced objects. In my entity copying for reports, I needed deep copy to avoid modifying cached data through references.
   - **If asked more:** I can explain Object.clone() doing shallow copy, how to implement deep copy via serialization or manual recursion, performance overhead of deep copy, and libraries like Apache Commons for cloning utilities.
46. What are wrapper classes?
   - **Answer:** Wrapper classes (Integer, Double, Boolean, etc.) box primitives into objects. I use them when collections require objects, like `Map<String, Integer>` for partner counts in inventory caching.
   - **If asked more:** I can explain autoboxing/unboxing, caching ranges (Integer cache -128 to 127), performance overhead of boxing in loops, comparison gotchas with `==`, and OptionalInt vs Optional<Integer>.
47. What is autoboxing and unboxing?
   - **Answer:** Autoboxing automatically converts primitives to wrapper objects, unboxing does the reverse. In my code, Java automatically converts when I put an `int` into a `List<Integer>` or use a wrapper in arithmetic.
   - **If asked more:** I can explain compiler-generated boxing/unboxing code, performance cost in tight loops, null pointer risks with unboxing null wrappers, and how to avoid pitfalls with primitive streams.
48. What are annotations?
   - **Answer:** Annotations are metadata tags added to code elements. In Spring Boot, I heavily use annotations like `@Service`, `@RestController`, `@Transactional`, and `@Cacheable` to declaratively configure behavior without XML.
   - **If asked more:** I can explain retention policies (SOURCE, CLASS, RUNTIME), target types, how Spring processes annotations via reflection, creating custom annotations for cross-cutting concerns, and meta-annotations.
49. What is reflection?
   - **Answer:** Reflection allows inspecting and invoking classes, methods, and fields at runtime. Spring Boot uses reflection heavily for dependency injection, but in my direct code I rarely use it — once for a dynamic field-mapping utility in my CDMS project.
   - **If asked more:** I can explain Class.forName(), getMethod/invoke, performance overhead, security restrictions with SecurityManager, how Spring minimizes reflection cost with caching, and alternatives like method handles.
50. What are generics?
   - **Answer:** Generics enable type-safe collections and classes by parameterizing types. In my projects, `List<SerialRecord>` ensures only SerialRecord objects are added, eliminating casting and catching type errors at compile time.
   - **If asked more:** I can explain type parameters, generic methods, bounded wildcards (? extends T / ? super T), how generics improve code reusability in Spring's JpaRepository<T, ID>, and the PECS principle.
51. What is type erasure?
   - **Answer:** Type erasure removes generic type information at runtime, so `List<String>` and `List<Integer>` both become just `List`. This means I cannot check generic types at runtime, which affected my reflection-based field mapper design.
   - **If asked more:** I can explain how javac replaces type parameters with bounds or Object, bridge methods, why you cannot create `new T()`, workarounds using TypeToken or Class<T> parameters, and implications for serialization.
52. What is varargs?
   - **Answer:** Varargs allow methods to accept variable number of arguments using `...` syntax. I used varargs in my validation framework to pass multiple error codes to a logging utility without overloading methods.
   - **If asked more:** I can explain the internal array creation, how varargs must be the last parameter, heap pollution warnings with generics, and when to avoid varargs for clarity.
53. What is enum?
   - **Answer:** Enums define a fixed set of named constants, and in Java they are full classes with fields and methods. In my cold-chain project, I used `AlertSeverity` enum (LOW, MEDIUM, HIGH, CRITICAL) with threshold values and action methods attached.
   - **If asked more:** I can explain enum singleton pattern, switch-case with enums, EnumSet/EnumMap for performance, when to use enums vs constants, and how JVM ensures enum instantiation safety against reflection.
54. What is garbage collection?
   - **Answer:** GC automatically reclaims memory from objects no longer reachable. In my Spring Boot apps, I rely on GC to clean up request-scoped objects and DTOs, but I had to tune heap settings during bulk inventory processing to avoid pauses.
   - **If asked more:** I can explain the mark-sweep-compact algorithm, generational collection (young/old), common collectors (G1, ZGC), how to monitor GC with JVM flags, and GC tuning for low-latency Kafka consumers.
55. What are GC roots?
   - **Answer:** GC roots are special objects from which the GC traces reachability, including active thread stacks, static fields, JNI references, and monitor locks. Objects not reachable from any root are candidates for collection.
   - **If asked more:** I can explain the root scanning process in GC, how it impacts pause times, common root categories, heap dump analysis tools (Eclipse MAT), and how memory leak analysis works by identifying unwanted references from roots.
56. What is memory leak in Java?
   - **Answer:** A memory leak occurs when objects are no longer needed but remain reachable. In my inventory project, I fixed a leak where cached serial record maps in a static HashMap were never cleared, causing heap growth over time.
   - **If asked more:** I can explain common leak patterns (unclosed streams, ThreadLocal misuse, inner class references, listener registrations), how to detect leaks via heap dumps, and weak references as cleanup tools.
57. How can memory leaks happen in Java?
   - **Answer:** Common causes include forgetting to close resources, holding objects in static collections, ThreadLocal not removed after use, JVM cached String intern, and unclosed streams. In my project, a static cache map caused gradual heap exhaustion.
   - **If asked more:** I can explain incident-driven cleanup with WeakHashMap, how JDBC connection leaks crash applications, profiling with VisualVM or JFR, and preventive patterns like try-with-resources and bounded caches.
58. What are strong, weak, soft, and phantom references?
   - **Answer:** Strong references prevent GC collection; soft references are collected before OOM (useful for caches); weak references are collected at next GC (used by WeakHashMap); phantom references track object finalization. I used WeakHashMap for temporary cache in cold-chain data aggregation.
   - **If asked more:** I can explain ReferenceQueue interaction, how WeakHashMap works internally for canonical mappings, soft reference as memory-sensitive cache, phantom reference for pre-mortem cleanup, and real use cases in frameworks like Guava cache.
59. What is classloader?
   - **Answer:** The classloader loads .class files into the JVM memory. In Spring Boot, the classloader handles loading from BOOT-INF/lib, which is why fat JARs work — I had to debug classloader issues when migrating from plain JAR to Spring Boot.
   - **If asked more:** I can explain the delegation model (Bootstrap -> Platform -> System -> custom), how custom classloaders enable hot deployment, Tomcat's per-webapp classloader, and common ClassNotFoundException troubleshooting.
60. What are Java records?
   - **Answer:** Records are concise data carriers introduced in Java 14. They automatically generate constructor, getters, equals, hashCode, and toString. In my DTO layers, I plan to use records for immutable transfer objects to reduce boilerplate code.
   - **If asked more:** I can explain canonical and compact constructors, restrictions (no extends, final fields), how records interact with JPA (issue with no-arg constructor), serialization behavior, and use cases for API response DTOs.

---

## Java Collections Questions

1. What is Java Collections Framework?
   - **Answer:** The Collections Framework provides interfaces and implementations for storing and manipulating groups of objects like List, Set, and Map. In my projects, I use ArrayList for ordered data, HashMap for lookups, and HashSet for deduplication of serial records in inventory validation.
   - **If asked more:** I can explain the hierarchy (Collection -> List/Set/Queue, Map separately), utility class Collections, how choosing wrong collection impacts performance, and when I prefer arrays over collections for primitive-heavy data.
2. Difference between `List`, `Set`, and `Map`.
   - **Answer:** List allows duplicates and ordered access, Set ensures uniqueness, and Map stores key-value pairs. In my inventory system, I used List for partner records, Set for unique serial numbers, and Map for partner-to-count cache.
   - **If asked more:** I can explain implementation differences (ArrayList vs LinkedList, HashSet vs TreeSet, HashMap vs TreeMap), ordering guarantees, null handling, and thread-safe variants like CopyOnWriteArrayList.
3. Difference between `ArrayList` and `LinkedList`.
   - **Answer:** ArrayList uses a dynamic array with O(1) random access, LinkedList uses doubly-linked nodes with O(n) access but O(1) insertion at ends. I always default to ArrayList because memory locality and cache performance are better for typical iteration patterns in my APIs.
   - **If asked more:** I can explain when LinkedList is actually useful (frequent front insertion, queue-like operations), memory overhead per element, how subList works with ArrayList, and why LinkedList is rarely the right choice in backend code.
4. Difference between `HashSet` and `TreeSet`.
   - **Answer:** HashSet uses hashCode for O(1) operations, TreeSet uses Red-Black tree for sorted order. I use HashSet for deduplication of serial records during validation, and would choose TreeSet only if I needed sorted iteration without extra sorting step.
   - **If asked more:** I can explain internal structures, how equals/hashCode must be consistent for HashSet, how TreeSet requires Comparable or Comparator, performance tradeoffs, and LinkedHashSet as a middle ground.
5. Difference between `HashMap` and `Hashtable`.
   - **Answer:** HashMap is not synchronized and allows null keys/values, while Hashtable is synchronized and does not allow nulls. In my projects, I always use HashMap and handle synchronization externally with ConcurrentHashMap when needed for thread safety.
   - **If asked more:** I can explain Hashtable's legacy status, how HashMap's default capacity and load factor differ, performance comparison, why ConcurrentHashMap replaced Hashtable, and iteration behavior differences.
6. Difference between `HashMap` and `ConcurrentHashMap`.
   - **Answer:** HashMap is not thread-safe, while ConcurrentHashMap uses internal locking for concurrent access without blocking reads. In my Kafka consumer where multiple threads update a shared cache, I used ConcurrentHashMap over HashMap with external sync.
   - **If asked more:** I can explain ConcurrentHashMap's bucket-level locking in Java 7 vs CAS-based approach in Java 8, how computeIfAbsent works atomically, performance scaling, and why Collections.synchronizedMap is less efficient.
7. Difference between `HashMap` and `LinkedHashMap`.
   - **Answer:** LinkedHashMap maintains insertion order (or access order) via a doubly-linked list, while HashMap does not guarantee order. I used LinkedHashMap in my cold-chain project when I needed to preserve the order of sensor readings as they arrived.
   - **If asked more:** I can explain how LinkedHashMap enables LRU cache with removeEldestEntry(), memory overhead of the linked list, performance vs HashMap, and iteration order guarantees.
8. Difference between `HashMap` and `TreeMap`.
   - **Answer:** TreeMap stores keys in sorted order using Red-Black tree, while HashMap is unordered. I would use TreeMap if I needed range queries or sorted iteration, like generating partner reports alphabetically sorted by partner name.
   - **If asked more:** I can explain O(log n) vs O(1) performance, how TreeMap implements NavigableMap for ceiling/floor/lower/higher methods, Comparable vs Comparator, and when TreeMap's sorting is unnecessary overhead.
9. How does `HashMap` work internally?
   - **Answer:** HashMap stores entries in an array of buckets. Each bucket is a linked list or tree. When I call put(key, value), HashMap computes key's hashCode, finds the bucket index, and places the entry there. On collision, entries chained in that bucket.
   - **If asked more:** I can explain hash function (hashing, bit-masking for index), treeification threshold (8 -> tree, 6 -> untree), initial capacity (16), load factor (0.75), rehashing logic, and how Java 8 improved collision handling with TreeNode.
10. What happens during hash collision?
   - **Answer:** When two keys have the same bucket index, HashMap stores both entries as a linked list in that bucket. In Java 8, if the list exceeds 8 entries, it converts to a balanced tree for O(log n) lookup instead of O(n).
   - **If asked more:** I can explain how equals() differentiates collided entries, the treeify threshold and untreeify threshold, how a bad hashCode implementation floods one bucket, and how to design keys to minimize collisions.
11. What is load factor?
   - **Answer:** Load factor (default 0.75) determines when HashMap resizes: when size exceeds capacity * load factor. A lower factor wastes memory but reduces collisions; higher factor saves memory but increases lookup time.
   - **If asked more:** I can explain how to choose initial capacity if I know the expected size (capacity = expected / load factor + 1), how resizing is expensive, and how I pre-sized HashMaps when caching 10,000+ inventory records.
12. What is rehashing?
   - **Answer:** Rehashing is the process of resizing the HashMap's bucket array and redistributing all entries when the load factor threshold is crossed. It involves creating a new array (2x size), recomputing bucket indices, and moving entries, which is O(n) and expensive.
   - **If asked more:** I can explain how Java 8 avoids rehashing all keys (just tests bit for new index), how concurrent resizing is avoided in ConcurrentHashMap, and why tuning initial capacity reduces rehashing overhead in large caches.
13. Why should keys be immutable in a `HashMap`?
   - **Answer:** Immutable keys guarantee hash code stability. If a key's hashCode changes after insertion, the HashMap cannot find it in the correct bucket, causing memory leaks and lookup failures. I always use String or Integer keys which are inherently immutable.
   - **If asked more:** I can explain precisely what happens when a mutable key's hash changes (the entry is orphaned), how to detect such issues in production, and how Java records make ideal keys with their built-in equals/hashCode.
14. What happens if a mutable object is used as a key?
   - **Answer:** If I modify a key after inserting it into a HashMap, the hash code changes but the entry stays in the original bucket. The entry becomes permanently lost — get() looks in the new bucket and finds nothing, while the old entry stays in memory.
   - **If asked more:** I can explain a real bug scenario I've seen where entity modification after map insertion caused data loss, how defensive copying helps, and how to use immutable wrappers or records to prevent this.
15. What is fail-fast iterator?
   - **Answer:** A fail-fast iterator throws ConcurrentModificationException if the collection is structurally modified after the iterator is created. In my single-threaded code, modifying a list while iterating with for-each triggers this, so I use Iterator.remove() instead.
   - **If asked more:** I can explain the modCount field mechanism, why fail-fast is a bug-detection feature not a guarantee, how to safely modify during iteration (CopyOnWriteArrayList or collect then remove), and race conditions in concurrent scenarios.
16. What is fail-safe iterator?
   - **Answer:** A fail-safe iterator works on a snapshot of the collection, so modifications after creation do not throw exceptions. Iterators of ConcurrentHashMap and CopyOnWriteArrayList are fail-safe, which I rely on when iterating shared caches in Kafka consumers.
   - **If asked more:** I can explain the snapshot vs live-data approach, how ConcurrentHashMap's iterator reflects some concurrent updates but not all, memory overhead of CopyOnWriteArrayList, and why fail-safe does not mean full thread safety.
17. Difference between `Iterator` and `ListIterator`.
   - **Answer:** ListIterator extends Iterator with bidirectional traversal, index access, and element modification/replacement. I use Iterator for general collection iteration and ListIterator when I need to traverse backwards or insert during iteration in my data processing.
   - **If asked more:** I can explain available methods (hasPrevious, previousIndex, set, add), which collections support ListIterator (List implementors), and why there is no SetIterator — Set has no positional access.
18. Difference between `Comparable` and `Comparator`.
   - **Answer:** Comparable defines natural ordering inside the class (compareTo), Comparator is external sorting logic. In my projects, I implement Comparable in entity classes for default sorting, and create custom Comparators for specific report ordering requirements.
   - **If asked more:** I can explain how compareTo contract mirrors equals consistency, lambda-based Comparator construction, Comparator.comparing() chaining, null handling with nullsFirst/nullsLast, and TreeSet/TreeMap requirements.
19. What is priority queue?
   - **Answer:** PriorityQueue orders elements by their natural order or a custom Comparator, not insertion order. The head is always the smallest element. I would use it for task scheduling scenarios, like processing inventory alerts by severity priority.
   - **If asked more:** I can explain heap-based implementation, O(log n) offer/poll, O(1) peek, why it is not thread-safe, PriorityBlockingQueue for concurrent use, and how Iterator does not guarantee priority order.
20. What is blocking queue?
   - **Answer:** BlockingQueue is a queue that blocks when taking from an empty queue or adding to a full queue. In my cold-chain project, we conceptually used this pattern for Kafka consumer message processing where consumers block until new sensor data arrives.
   - **If asked more:** I can explain implementations like ArrayBlockingQueue (bounded, fair), LinkedBlockingQueue, how producers/consumers coordinate, delay queues for scheduled processing, and how thread pools internally use BlockingQueue.
21. What is copy-on-write collection?
   - **Answer:** CopyOnWriteArrayList creates a new underlying array on every modification, so iterators never see stale data and never throw ConcurrentModificationException. I would use it for read-heavy, write-rare scenarios like configuration lists read by multiple threads.
   - **If asked more:** I can explain why it is expensive for writes (O(n) array copy), snapshot iterator semantics, memory implications, how it differs from ConcurrentHashMap's design, and when the cost is justified (frequent reads, rare writes).
22. When would you use `ArrayList`?
   - **Answer:** I use ArrayList almost always for ordered data because of O(1) random access, cache-friendly memory layout, and no per-element memory overhead. In my inventory system, I store lists of serial records in ArrayList for fast indexed access during validation.
   - **If asked more:** I can explain the dynamic growth (newSize = old + old >> 1), why toArray() with size zero is fastest, subList view behavior, and when to use Arrays.asList vs ArrayList constructor.
23. When would you use `LinkedList`?
   - **Answer:** I rarely use LinkedList in backend code. It makes sense when I need frequent insertions at both ends (deque operations) and do not need random access. Even then, ArrayDeque usually outperforms LinkedList for stack/queue use cases in my experience.
   - **If asked more:** I can explain why LinkedList has poor cache locality, higher memory per element (2 references + object header), how it implements both List and Deque, and why ArrayList + ArrayDeque cover 99% of backend needs.
24. When would you use `ConcurrentHashMap`?
   - **Answer:** I use ConcurrentHashMap whenever a HashMap is accessed by multiple threads, such as my cache of partner serial counts that was read/updated by multiple Kafka consumer threads. It gives me thread safety without explicit synchronization overhead.
   - **If asked more:** I can explain the CAS + synchronized segment design in Java 8, why computeIfAbsent is atomic, how to safely iterate and update concurrently, when to use ConcurrentHashMap vs Collections.synchronizedMap, and performance tradeoffs.
25. When would you use `TreeMap`?
   - **Answer:** I use TreeMap when I need keys sorted automatically, like maintaining a time-sorted map of sensor events by timestamp for dashboard display. TreeMap's NavigableMap features (ceilingKey, subMap) are useful for range queries on sorted data.
   - **If asked more:** I can explain O(log n) performance for all operations, memory overhead of tree nodes, when to prefer TreeMap over sorting HashMap entries after retrieval, and how ConcurrentSkipListMap is the thread-safe alternative.

---

## Java 8 and Modern Java Questions

1. What are Java 8 features?
   - **Answer:** Java 8 introduced lambdas, Stream API, Optional, default methods, CompletableFuture, and the new Date/Time API. In my projects, I use lambdas and streams daily for collection processing — filtering partners, mapping DTOs, and collecting reports without loops.
   - **If asked more:** I can explain the motivation behind each feature, how lambdas enabled functional-style programming, why the Date/Time API replaced Date/Calendar, and which features I use most (Stream API and Optional).
2. What is lambda expression?
   - **Answer:** Lambda expressions let me pass behavior as a method argument concisely. In my cold-chain project, I used lambdas in Stream API to filter sensor readings above a threshold and map them to alert objects without writing verbose anonymous classes.
   - **If asked more:** I can explain lambda syntax (parameters -> body), type inference by the compiler, variable capture (effectively final), how lambdas are compiled to invokedynamic, and common pitfalls like modifying captured variables.
3. What is functional interface?
   - **Answer:** A functional interface has exactly one abstract method and can be implemented by a lambda. Built-in ones like Predicate, Function, Consumer, and Supplier are used heavily in Stream API operations in my data processing pipelines.
   - **If asked more:** I can explain how to create custom functional interfaces, how @FunctionalInterface enforces the contract, how default methods do not break functional interface status, and examples like Comparator with lambda.
4. What is `@FunctionalInterface`?
   - **Answer:** @FunctionalInterface is an annotation that marks an interface as intended for lambda use, and the compiler enforces exactly one abstract method. I use it when defining custom functional interfaces for specific transformation logic in my validation framework.
   - **If asked more:** I can explain how overriding equals from Object does not count as abstract, what happens if multiple abstract methods exist (compilation error), and how Runnable and Callable are functional interfaces.
5. What is Stream API?
   - **Answer:** Stream API processes sequences of data declaratively using functional operations. In my inventory project, I used streams to filter, sort, and collect serial records by partner, replacing complex for-loops with readable one-liners.
   - **If asked more:** I can explain pipelines, intermediate vs terminal operations, stream sources (collections, arrays, I/O, generate/iterate), how streams do not modify the source, and Collectors.toMap/groupingBy for aggregation.
6. Difference between collection and stream.
   - **Answer:** Collections store all elements in memory, while streams compute elements on demand and cannot be reused. I use collections as data storage and streams for transformation pipelines — the data stays in the collection, streams just process it once.
   - **If asked more:** I can explain how collections are about data, streams are about computation, internal vs external iteration, why streams can be parallelized easily, and how a stream after terminal operation is consumed.
7. Difference between intermediate and terminal operations.
   - **Answer:** Intermediate operations (filter, map, sorted) return a new stream and are lazy, terminal operations (collect, forEach, count) trigger processing. In my filter-and-collect pattern, filter and map just build a pipeline until collect executes everything.
   - **If asked more:** I can explain how intermediate operations are fused, why order matters (filter first, then map), example of short-circuiting terminal operations (findFirst, anyMatch), and how peek differs from forEach.
8. What is lazy evaluation in streams?
   - **Answer:** Lazy evaluation means intermediate operations are not executed until a terminal operation is invoked. In my code, chaining multiple filters on a large collection does not process elements until collect() is called, enabling optimization like operation fusion.
   - **If asked more:** I can explain how laziness enables infinite streams (Stream.generate, Stream.iterate), how the JVM optimizes the pipeline, debugging with peek, and why lazy evaluation improves performance for early-terminating operations.
9. Difference between `map()` and `flatMap()`.
   - **Answer:** map transforms each element 1-to-1, flatMap transforms 1-to-many and flattens the result. I use map for simple DTO conversion (Entity->ResponseDTO) and flatMap when each sensor reading produces multiple alert objects in my cold-chain project.
   - **If asked more:** I can explain how flatMap works with nested collections, Optional.flatMap for chaining optional operations, flatMapping with Collectors.groupingBy for multi-level aggregation, and how flatMap differs in stream vs Optional.
10. Difference between `filter()` and `map()`.
   - **Answer:** filter selects elements matching a predicate (narrowing), map transforms each element (changing). I chain filter before map to reduce processing — e.g., filter valid serial records then map to response DTOs in inventory APIs.
   - **If asked more:** I can explain how filter uses Predicate, map uses Function, how combining filter+map is idiomatic, why placing filter first reduces downstream work, and how distinct() and limit() are also filtering operations.
11. Difference between `findFirst()` and `findAny()`.
   - **Answer:** findFirst returns the first element in encounter order, findAny returns any element non-deterministically. In sequential streams they behave identically, but findAny is optimized for parallel streams because it does not enforce ordering.
   - **If asked more:** I can explain how encounter order depends on the source (List vs Set vs HashSet), why findFirst is slower in parallel, real use cases like "find any valid partner" vs "find the earliest alert", and orElseThrow usage.
12. Difference between sequential and parallel streams.
   - **Answer:** Sequential streams process elements in a single thread, parallel streams split work across multiple threads using ForkJoinPool. I avoid parallel streams in my projects because most of my collections are small or involve I/O where parallel adds overhead.
   - **If asked more:** I can explain the common ForkJoinPool, how parallelism works with spliterator, when parallel streams help (large datasets, CPU-intensive operations), and pitfalls like shared mutable state or blocking operations.
13. When should you avoid parallel streams?
   - **Answer:** I avoid parallel streams for small datasets, I/O-bound operations (DB calls, HTTP), operations with shared mutable state, and ordered streams where merge overhead outweighs gain. My inventory collections of 10,000 records were too small to benefit.
   - **If asked more:** I can explain how parallel overhead includes splitting, merging, thread coordination, why the data must be large (100k+ elements) to benefit, how to measure with JMH, and how n/parallelism determines speedup.
14. What is `Optional`?
   - **Answer:** Optional is a container that may or may not hold a value, used to avoid null pointer exceptions. In my CDMS APIs, I used Optional when fetching partner data by ID from JpaRepository, then handled present/empty cases with orElseThrow.
   - **If asked more:** I can explain how Optional encourages explicit null handling, why it is not for fields or method parameters (only return types), common methods (map, flatMap, filter, ifPresent), and performance overhead vs direct null check.
15. Why should we not use `Optional.get()` directly?
   - **Answer:** Optional.get() throws NoSuchElementException if the Optional is empty, defeating the purpose of Optional. In my code, I use orElse(), orElseThrow(), or orElseGet() to provide safe default values or custom exceptions.
   - **If asked more:** I can explain how orElseThrow is preferred for "must exist" cases, how orElseGet avoids eager evaluation, why isPresent()+get() is an anti-pattern (use ifPresent or map instead), and how IntelliJ warns about direct get().
16. Difference between `orElse()` and `orElseGet()`.
   - **Answer:** orElse always evaluates the default value even if the Optional is present, while orElseGet takes a Supplier that runs only when the Optional is empty. In performance-sensitive code, I use orElseGet to avoid unnecessary object creation.
   - **If asked more:** I can explain why orElse with method call (orElse(computeDefault())) executes computeDefault always, a real bug I encountered with expensive default computation, and how orElseThrow bridges Optional with custom exceptions.
17. What are method references?
   - **Answer:** Method references provide shorter syntax for lambdas that call an existing method (ClassName::methodName). I use them in stream pipelines — e.g., `.map(PartnerDTO::new)` for constructor references or `.forEach(logger::info)` for method calls.
   - **If asked more:** I can explain the four types: static (Integer::parseInt), instance (String::toLowerCase), constructor (ArrayList::new), and arbitrary object (String::length), and when to choose method references over lambdas for readability.
18. What are default methods?
   - **Answer:** Default methods allow interfaces to have method implementations without breaking implementing classes. I use them when extending functional interfaces with convenience methods, like adding a default `orElseThrow` method to a custom validation interface.
   - **If asked more:** I can explain why they were introduced (Java 8 streams needed Collection.forEach without breaking existing code), how conflicts are resolved (class wins over interface), and the diamond problem with multiple default methods.
19. What is `CompletableFuture`?
   - **Answer:** CompletableFuture is a Future that can be manually completed and supports chaining async operations. In my cold-chain project, I used CompletableFuture to fetch sensor data from multiple IoT sources asynchronously and combine results without blocking.
   - **If asked more:** I can explain how it extends Future, the async callback chaining (thenApply, thenCompose), how supplyAsync/runAsync create async tasks, how I combined multiple futures with allOf, and custom thread pool configuration.
20. Difference between `thenApply()` and `thenCompose()`.
   - **Answer:** thenApply transforms the result of a CompletableFuture synchronously (returns a nested CompletableFuture if the function returns one), while thenCompose flattens nested futures. I use thenApply for simple transformations and thenCompose to chain dependent async calls.
   - **If asked more:** I can explain how thenApply is like Stream.map and thenCompose is like Stream.flatMap, the issue of CompletableFuture<CompletableFuture<>> with thenApply, and how thenCompose avoids callback hell.
21. Difference between `thenApply()` and `thenAccept()`.
   - **Answer:** thenApply transforms a value and returns a result, thenAccept consumes the value and returns Void. I use thenAccept when I need to perform a side effect (like logging or caching) after a future completes without returning a new value.
   - **If asked more:** I can explain how each method relates to Function vs Consumer, how thenRun works for Runnable, use cases like thenAccept for sending notifications after async processing, and exception propagation.
22. How do you handle exceptions in `CompletableFuture`?
   - **Answer:** I use exceptionally() to recover from errors with a fallback value, handle() to process both success and failure, and whenComplete for side-effect cleanup. In my async IoT data pipeline, I logged errors via exceptionally and used fallback readings.
   - **If asked more:** I can explain how exceptions in one stage propagate to downstream stages, how handle() differs from exceptionally (handle always runs), how completeExceptionally is used, and why unchecked exceptions need careful handling.
23. What is Java Date and Time API?
   - **Answer:** The java.time package provides immutable, thread-safe date/time classes like LocalDate, LocalDateTime, ZonedDateTime, and Instant. I use LocalDate for inventory dates and Instant for sensor timestamps in cold-chain, replacing the flawed java.util.Date.
   - **If asked more:** I can explain the problems with legacy Date (mutable, poor design, month=0), how Duration/Period measure time, how DateTimeFormatter replaces SimpleDateFormat, and timezone handling with ZoneId.
24. Difference between `LocalDateTime`, `ZonedDateTime`, and `Instant`.
   - **Answer:** LocalDateTime has no timezone, ZonedDateTime includes full timezone rules, and Instant is a UTC timestamp. I use Instant for storing IoT sensor timestamps in InfluxDB, ZonedDateTime for user-facing displays, and LocalDate for inventory date-only fields.
   - **If asked more:** I can explain how to convert between them, why LocalDateTime should not be stored without knowing the zone, daylight saving handling in ZonedDateTime, and OffsetDateTime as a lighter alternative to ZonedDateTime.
25. What are records in Java?
   - **Answer:** Records are immutable data carriers introduced in Java 14 that generate constructor, getters, equals, hashCode, and toString automatically. In my DTO layers, I use records for API response objects to reduce boilerplate and ensure immutability.
   - **If asked more:** I can explain canonical constructor, compact constructor for validation, how records are final and extend java.lang.Record, serialization behavior, JPA limitations (no-arg constructor missing), and pattern matching with records in Java 16+.
26. What are sealed classes?
   - **Answer:** Sealed classes restrict which classes can extend them, providing controlled inheritance. I would use sealed classes for domain event types in my projects — like `SealedEvent permits SensorEvent, AlertEvent, SystemEvent` — for exhaustive pattern matching.
   - **If asked more:** I can explain permits clause, sealed interfaces as well, how exhaustive switch (Java 17+) works with sealed types, the `non-sealed` modifier, and how they improve domain modeling compared to final or unrestricted classes.
27. What is pattern matching?
   - **Answer:** Pattern matching allows type checking and deconstruction in a single construct. In Java 16+, I can use pattern matching for instanceof: `if (obj instanceof String s)` which eliminates separate casting, making my validation code cleaner.
   - **If asked more:** I can explain how it evolved across versions (instanceof in 16, switch in 17+, records in 19+), guard patterns with `&&`, exhaustive matching with sealed classes, and how it reduces boilerplate in if-else chains.
28. What are switch expressions?
   - **Answer:** Switch expressions return a value and use arrow syntax for concise cases. I use switch expressions when mapping enum values to strings or numeric thresholds, like converting `AlertSeverity.LOW` to a color code without break statements.
   - **If asked more:** I can explain arrow vs colon syntax, why no fall-through with arrows, yield keyword for blocks, exhaustive requirements (or default needed), and pattern matching in switch from Java 17+ for richer conditions.
29. What are text blocks?
   - **Answer:** Text blocks (""") provide multi-line string literals with clean formatting. I use them for SQL queries in my inventory project — embedding multi-line validation queries without concatenation or escape sequences for newlines.
   - **If asked more:** I can explain how leading whitespace is stripped using indentation alignment, escape sequences within text blocks, how they improve readability of JSON/HTML/SQL strings, and the equivalent Java 13 preview.
30. What are virtual threads?
   - **Answer:** Virtual threads are lightweight threads from Project Loom (Java 21+) that are managed by the JVM, not the OS, allowing millions of concurrent tasks. I am excited to use them for handling high-volume Kafka message processing where each sensor reading can be a virtual thread.
   - **If asked more:** I can explain how virtual threads differ from platform threads, when to use (I/O-heavy, many concurrent tasks) vs avoid (CPU-bound), how they work with synchronized, the Executors.newVirtualThreadPerTaskExecutor(), and Spring Boot 3.2+ virtual thread support.

---

## Java Concurrency Questions

1. What is a thread?
   - **Answer:** A thread is the smallest unit of execution within a process. Each thread has its own stack but shares heap memory. In my Kafka consumers, each listener thread processes messages concurrently, sharing access to caches while maintaining independent execution.
   - **If asked more:** I can explain thread lifecycle (NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED), how threads map to OS threads in platform threads vs virtual threads, and thread stack size tuning.
2. What is process vs thread?
   - **Answer:** A process is an independent program with its own memory space, while threads share memory within a process. Communication between processes requires IPC (pipes, sockets), but threads can communicate via shared objects, which is why I use threads for concurrent data processing.
   - **If asked more:** I can explain context switching costs (threads are lighter), how the JVM runs as a single process with multiple threads, and why multithreading is preferred over multiprocessing for backend services.
3. How do you create a thread in Java?
   - **Answer:** I can extend Thread, implement Runnable, or use Callable with ExecutorService. In my projects, I never extend Thread directly — I use Runnable with thread pools or CompletableFuture.supplyAsync for cleaner async execution.
   - **If asked more:** I can explain the different creation approaches, why implementing Runnable is better than extending Thread (flexibility, no single-inheritance limitation), and how lambda syntax simplifies new Thread(() -> work()).start().
4. Difference between extending `Thread` and implementing `Runnable`.
   - **Answer:** Extending Thread locks you into inheritance; implementing Runnable leaves the class free to extend other classes. In my Spring services, I always implement Runnable or use lambdas, keeping the class available for Spring bean proxying.
   - **If asked more:** I can explain how Thread itself implements Runnable, why extending Thread is considered an anti-pattern, the strategy pattern benefit of decoupling task from execution, and how ExecutorService accepts Runnable.
5. What is `Callable`?
   - **Answer:** Callable is like Runnable but can return a result and throw checked exceptions. In my inventory project, I used Callable with ExecutorService to submit validation tasks that returned counts of invalid serial records.
   - **If asked more:** I can explain the call() method vs run(), how Future wraps the result, why Callable is preferred for tasks needing return values, and how CompletableFuture.supplyAsync internally uses Callable.
6. Difference between `Runnable` and `Callable`.
   - **Answer:** Runnable returns void and cannot throw checked exceptions; Callable returns a value and can throw Exception. I use Runnable for fire-and-forget tasks like logging and Callable when I need a result from async computation.
   - **If asked more:** I can explain how ExecutorService.submit(Runnable) returns Future<?> vs submit(Callable) returns Future<T>, how to get null from Runnable futures, and lambda syntax (() -> result vs () -> { run(); }).
7. What is `Future`?
   - **Answer:** Future represents the result of an async computation with methods like get(), isDone(), and cancel(). In my projects, I moved to CompletableFuture for chaining, but Future was my earlier approach for getting results from thread pool tasks.
   - **If asked more:** I can explain blocking get() vs timeout version, how cancel works with mayInterruptIfRunning, limitations of Future (no chaining, no manual completion, no callbacks), and how CompletableFuture improves on it.
8. What is `ExecutorService`?
   - **Answer:** ExecutorService manages a pool of threads and decouples task submission from execution. In my CDMS project, I configured a custom ExecutorService for parallel report generation, submitting tasks and collecting results via invokeAll.
   - **If asked more:** I can explain ThreadPoolExecutor parameters (corePoolSize, maxPoolSize, keepAliveTime, workQueue), different factory methods (newFixedThreadPool, newCachedThreadPool), and shutdown vs shutdownNow.
9. What is thread pool?
   - **Answer:** A thread pool reuses a fixed number of threads to execute tasks, avoiding the overhead of creating new threads per request. In my Spring Boot async configuration, I defined a thread pool for @Async methods to handle concurrent inventory processing.
   - **If asked more:** I can explain how thread pools improve performance (reuse, control resource usage), the worker thread pattern, common pool types (fixed, cached, scheduled, single), and how rejecting tasks with RejectedExecutionHandler works.
10. Why use thread pools?
   - **Answer:** Thread pools reduce overhead from thread creation, control concurrency limits, and improve system stability. In my inventory project, using a bounded thread pool prevented runaway threads during peak validation from overwhelming the database connection pool.
   - **If asked more:** I can explain the cost of creating threads (memory, OS handles), how thread pools smooth out burst loads, thread starvation vs resource exhaustion, and monitoring thread pool metrics via JMX.
11. Difference between fixed thread pool and cached thread pool.
   - **Answer:** Fixed thread pool keeps a constant number of threads; cached thread pool creates new threads as needed and reuses idle ones. I use fixed pools for controlled database access and avoid cached pools in production due to unbounded thread creation risk.
   - **If asked more:** I can explain the work queue difference (fixed uses LinkedBlockingQueue, cached uses SynchronousQueue), when cached is useful (many short-lived tasks), and why cached pools can crash a system under load.
12. What is scheduled executor service?
   - **Answer:** ScheduledExecutorService can run tasks with delay or periodically. In my projects, I used it for scheduled tasks like refreshing partner cache every 5 minutes, with scheduleAtFixedRate for consistent interval execution.
   - **If asked more:** I can explain schedule vs scheduleAtFixedRate vs scheduleWithFixedDelay, how missed executions are handled, ScheduledThreadPoolExecutor internals (DelayedWorkQueue), and Spring's @Scheduled abstraction over it.
13. What is synchronization?
   - **Answer:** Synchronization prevents multiple threads from executing critical sections simultaneously, ensuring thread safety via mutual exclusion. In my concurrent cache access, synchronized methods on shared maps prevented race conditions during inventory updates.
   - **If asked more:** I can explain intrinsic locks, the synchronized keyword on methods vs blocks, reentrancy, how synchronization creates happens-before guarantees, and performance costs of excessive synchronization.
14. What is race condition?
   - **Answer:** A race condition occurs when multiple threads access shared data simultaneously and the outcome depends on thread scheduling order. In my Kafka consumer, a race condition on shared counter caused incorrect record counts before I added synchronization.
   - **If asked more:** I can explain check-then-act and read-modify-write patterns, how to detect race conditions via code review, why atomic classes and locks prevent them, and how ConcurrentHashMap internal design avoids races.
15. What is deadlock?
   - **Answer:** Deadlock occurs when two or more threads each hold a lock and wait for the other to release, causing indefinite blocking. While I never encountered it in my projects, I design my locking order consistently (same order for all threads) to prevent it.
   - **If asked more:** I can explain the four necessary conditions (mutual exclusion, hold-and-wait, no preemption, circular wait), how to detect with jstack or visualvm, and prevention through lock ordering, timeouts, and tryLock.
16. How do you prevent deadlock?
   - **Answer:** I prevent deadlock by acquiring locks in a consistent global order, using tryLock with timeouts instead of intrinsic locks, and keeping critical sections as small as possible. In my cache design, a single ConcurrentHashMap eliminated the need for multiple locks.
   - **If asked more:** I can explain lock ordering strategy, how ReentrantLock.tryLock(time, unit) avoids indefinite blocking, deadlock detection via thread dumps, and how lock-free data structures bypass the problem entirely.
17. What is livelock?
   - **Answer:** Livelock is when threads keep changing state in response to each other without making progress, like two people stepping aside in the same direction repeatedly. I avoid it by adding random delays or backoff in retry logic.
   - **If asked more:** I can compare livelock with deadlock (blocked vs active-but-no-progress), real examples like two threads releasing and retrying locks in a loop, and how exponential backoff helps with coordination.
18. What is starvation?
   - **Answer:** Starvation occurs when a thread is perpetually denied access to resources because other threads keep getting priority. I ensure fairness in my thread pools by using a fair lock (new ReentrantLock(true)) when multiple threads contend for shared resources.
   - **If asked more:** I can explain how low-priority threads can starve, how synchronized blocks are unfair by default, how ReentrantLock allows fairness parameter, and how thread pool sizing affects starvation risk.
19. What is `volatile`?
   - **Answer:** volatile ensures that reads and writes to a variable are directly from main memory, not thread-local cache, providing visibility guarantees. In my Kafka consumer, I used a volatile boolean flag for graceful shutdown signal across threads.
   - **If asked more:** I can explain the happens-before relationship with volatile, why volatile does not provide atomicity (increment is still three operations), how it differs from synchronized, and common use cases (flags, double-checked locking).
20. Difference between `volatile` and `synchronized`.
   - **Answer:** volatile guarantees visibility only, synchronized guarantees both visibility and atomicity. I use volatile for simple flag variables and synchronized (or locks) for compound operations like check-then-act sequences.
   - **If asked more:** I can explain how synchronized provides mutual exclusion while volatile does not, memory barrier effects, how atomic classes like AtomicBoolean combine both, and the classic double-checked locking pattern with volatile.
21. What is atomic variable?
   - **Answer:** Atomic variables (AtomicInteger, AtomicLong, AtomicReference) support lock-free, thread-safe operations using CAS (Compare-And-Swap). In my inventory project, I used AtomicLong for tracking total processed records across consumer threads without locks.
   - **If asked more:** I can explain how CAS works (compareAndSet), why atomic variables are faster than locks for simple counters, the ABA problem with AtomicReference, and how LongAdder improves on AtomicLong for high-contention writes.
22. What is `AtomicInteger`?
   - **Answer:** AtomicInteger provides atomic operations like incrementAndGet, addAndGet, and compareAndSet for int values. In my cold-chain project, I used it to count total sensor events processed across multiple Kafka partitions without synchronized blocks.
   - **If asked more:** I can explain how incrementAndGet uses CAS in a loop, why it is thread-safe, how to use updateAndGet for custom transformations, and performance comparison with synchronized int.
23. What is lock?
   - **Answer:** A lock is a synchronization mechanism that provides more flexibility than synchronized blocks. In Java, ReentrantLock offers tryLock, timed lock, and interruptible lock acquisition. I prefer synchronized for simple cases and ReentrantLock for advanced scenarios.
   - **If asked more:** I can explain the Lock interface (lock, unlock, tryLock, lockInterruptibly), how lock() must be paired with unlock() in finally, condition variables with await/signal, and ReentrantLock fairness.
24. Difference between intrinsic lock and `ReentrantLock`.
   - **Answer:** Intrinsic locks (synchronized) are simpler and automatically released; ReentrantLock offers tryLock, fairness, and condition support. I use synchronized by default and ReentrantLock only when I need timed lock attempts or interruptible locking.
   - **If asked more:** I can explain how synchronized uses monitor enter/exit bytecodes, ReentrantLock uses AbstractQueuedSynchronizer, performance comparison (synchronized optimized in recent Java), and how tryLock helps prevent deadlocks.
25. What is read-write lock?
   - **Answer:** ReadWriteLock allows multiple readers simultaneously but exclusive write access. In my hot-cache scenario, I could use ReentrantReadWriteLock for a configuration map where reads are frequent and writes are rare, improving throughput.
   - **If asked more:** I can explain the readLock/writeLock split, how multiple readers do not block each other, write starvation with many readers, and when ReadWriteLock is beneficial vs when a simple ConcurrentHashMap suffices.
26. What is semaphore?
   - **Answer:** Semaphore controls access to a limited resource by maintaining a permit count. In my cold-chain project, I would use Semaphore to limit concurrent database connections during bulk sensor data writes, ensuring the connection pool is not exhausted.
   - **If asked more:** I can explain acquire() and release() methods, counting vs binary semaphore, how Semaphore differs from locks (no ownership), fair vs unfair acquisition, and practical use cases like rate limiting.
27. What is countdown latch?
   - **Answer:** CountDownLatch lets one or more threads wait until a set of operations complete. I could use it in my inventory system to wait for multiple parallel validation tasks to finish before proceeding to the aggregation step.
   - **If asked more:** I can explain how await() blocks until count reaches zero, how countDown() decrements, why CountDownLatch is single-use (cannot reset), and CyclicBarrier vs CountDownLatch differences.
28. What is cyclic barrier?
   - **Answer:** CyclicBarrier lets multiple threads wait for each other at a common point before proceeding. Unlike CountDownLatch, it can be reused. I would use it in batch processing where N worker threads must sync after each batch of records.
   - **If asked more:** I can explain barrier action (Runnable that runs when barrier trips), party count, how reset() makes it reusable, difference from CountDownLatch (threads wait for each other vs threads wait for countdown), and timeout usage.
29. What is concurrent collection?
   - **Answer:** Concurrent collections are thread-safe collections optimized for multi-threaded access, like ConcurrentHashMap, CopyOnWriteArrayList, and BlockingQueue. I use ConcurrentHashMap extensively in my Kafka consumers for shared caches without external synchronization.
   - **If asked more:** I can explain how ConcurrentHashMap uses CAS and synchronized internally, why CopyOnWriteArrayList is for read-heavy workloads, how BlockingQueue coordinates producers/consumers, and when to choose concurrent over synchronized wrappers.
30. What is `ThreadLocal`?
   - **Answer:** ThreadLocal provides per-thread variable instances. In Spring Boot, each HTTP request runs on a thread and ThreadLocal stores user context. I used ThreadLocal for request-scoped logging correlation IDs in my APIs.
   - **If asked more:** I can explain how ThreadLocalMap works internally (weak reference to thread), memory leak risks with thread pools (threads are reused, entries not cleaned up), and why remove() should always be called in async or pool scenarios.
31. How does Spring Security use `ThreadLocal`?
   - **Answer:** Spring Security stores the authenticated user in SecurityContextHolder using ThreadLocal by default, so every method in the same thread can access the current user's authentication. In my controllers, I get the logged-in user via SecurityContextHolder.getContext().
   - **If asked more:** I can explain MODE_THREADLOCAL (default per request thread), how it breaks in async execution, how to propagate context to child threads with MODE_INHERITABLETHREADLOCAL, and custom context propagation strategies.
32. What happens to `ThreadLocal` in async execution?
   - **Answer:** ThreadLocal values are not automatically propagated to async threads because the async task runs on a different thread. In my @Async methods, I had to manually pass context or use Spring's TaskDecorator to copy ThreadLocal values to the executing thread.
   - **If asked more:** I can explain how InheritableThreadLocal works for child threads but not thread pools, how to implement AsyncConfigurer with ThreadPoolTaskExecutor and TaskDecorator, and how MDC context is similarly lost in async.
33. What is context propagation?
   - **Answer:** Context propagation transfers thread-local state (security, tracing, MDC) across thread boundaries. In my Kafka consumers, I propagated MDC context from the listener thread to async processing threads so correlation IDs remained consistent across logs.
   - **If asked more:** I can explain challenges in thread pools, how libraries like Micrometer handle propagation, Spring Cloud Sleuth's approach with TraceRunnable, and OpenTelemetry's context propagation across services.
34. What is thread safety?
   - **Answer:** A class is thread-safe when it behaves correctly under concurrent access. In my design, I achieve thread safety through immutable objects, synchronized blocks, atomic variables, or thread-safe collections like ConcurrentHashMap.
   - **If asked more:** I can explain the four strategies: confinement (no sharing), immutability, synchronization, and thread-safe data structures, how to reason about thread safety with happens-before, and how to test with stress tests.
35. How do you make a class thread-safe?
   - **Answer:** I make a class thread-safe by using immutable fields (final), atomic classes for counters, synchronized blocks for critical sections, or delegating to ConcurrentHashMap/CopyOnWriteArrayList. In my cache service, I used ConcurrentHashMap with AtomicLong counters.
   - **If asked more:** I can explain identifying shared mutable state as the first step, choosing the right approach based on contention level (immutable > atomic > lock > synchronized), documenting thread-safety guarantees, and testing with multiple threads.

---

## Spring Boot Questions

1. What is Spring Framework?
   - **Answer:** Spring Framework is a lightweight, modular framework for building Java enterprise applications using IoC and dependency injection. In my CDMS project at Talentpace, I used Spring to manage service layers, repository injection, and transaction boundaries for MSSQL operations.
   - **If asked more:** I would explain the modular architecture: Core Container, Data Access, Web, AOP, and how I chose specific modules for my REST APIs instead of pulling the entire framework.
2. What is Spring Boot?
   - **Answer:** Spring Boot is Spring's opinionated auto-configuration layer that removes boilerplate setup. In my projects, I just added `spring-boot-starter-web`, `spring-boot-starter-data-jpa`, and `spring-boot-starter-security`, and everything was pre-configured for Tomcat, Hibernate, and security defaults.
   - **If asked more:** I would explain how `@SpringBootApplication` combines `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`, and how I used these annotations across my CDMS and inventory microservices.
3. Why use Spring Boot?
   - **Answer:** Spring Boot cuts development time by providing embedded Tomcat, auto-configuration, production-ready features like Actuator, and easy externalized configuration. In my inventory project, I deployed a Spring Boot JAR on EC2 and didn't need to manage a separate Tomcat install.
   - **If asked more:** I would explain the concrete time savings: I could go from a new starter project to a deployed API with JWT security and JPA in under an hour, which was critical for fast client demos.
4. Difference between Spring and Spring Boot.
   - **Answer:** Spring is the core framework providing DI, AOP, and MVC; Spring Boot adds auto-configuration, embedded servers, and starter dependencies on top. For my cold-chain APIs, I used Spring Boot so I didn't need to manually configure DispatcherServlet or Hibernate — Boot handled that automatically.
   - **If asked more:** I would explain that Spring Boot still uses Spring under the hood but adds `spring.factories` auto-configuration classes, and I'd show how I customized them with `application.yml` properties in my projects.
5. What is auto-configuration?
   - **Answer:** Auto-configuration is Spring Boot's ability to automatically configure beans based on dependencies in the classpath. For example, adding `spring-boot-starter-data-jpa` automatically configured my MSSQL DataSource, EntityManager, and TransactionManager in the inventory project.
   - **If asked more:** I would explain how `@ConditionalOnClass`, `@ConditionalOnMissingBean`, and `@ConditionalOnProperty` work internally, and how I used these to conditionally configure Redis cache only when the Redis dependency was present.
6. How does Spring Boot auto-configuration work?
   - **Answer:** Spring Boot scans `META-INF/spring.factories` for `EnableAutoConfiguration` classes and applies `@Conditional` checks to decide which beans to create. For my CDMS project, this meant when I added `spring-boot-starter-web`, Boot automatically configured Jackson, DispatcherServlet, and error handling without any XML.
   - **If asked more:** I would explain the `AutoConfigurationImportSelector` mechanism, how `@Conditional` prevents conflicting beans, and how I debugged auto-configuration using `--debug` flag and Actuator's `/conditions` endpoint.
7. What is starter dependency?
   - **Answer:** A starter dependency is a curated Maven POM that bundles related libraries. For my inventory validation API, I used `spring-boot-starter-validation` which brought in Hibernate Validator and its transitive dependencies, so I could use `@NotBlank` and `@Pattern` on DTOs immediately.
   - **If asked more:** I would explain how starters follow the naming convention `spring-boot-starter-*`, and how to create a custom starter by defining auto-configuration classes and a `spring.factories` file.
8. What is embedded server?
   - **Answer:** An embedded server is a web server bundled inside the application JAR. My Spring Boot JARs for CDMS and cold-chain projects ran on embedded Tomcat, which I configured by setting `server.port`, `server.servlet.context-path`, and SSL properties in `application.yml`.
   - **If asked more:** I would explain the difference between Tomcat, Jetty, and Undertow, and when I switched to Undertow for better performance in the cold-chain project that handled high-frequency IoT API calls.
9. What is IoC?
   - **Answer:** Inversion of Control means the framework controls object creation and lifecycle instead of the developer. In all my Talentpace projects, the Spring IoC container managed my service, repository, and security filter beans — I just defined them and injected where needed.
   - **If asked more:** I would explain the IoC container types — BeanFactory vs ApplicationContext — and how I used `ApplicationContext.getBean()` in a rare case to dynamically fetch cache manager beans based on tenant config.
10. What is dependency injection?
   - **Answer:** Dependency Injection is when Spring provides required objects instead of the class creating them. In my CDMS project, I used constructor injection for service and repository dependencies, which made testing easier with mocks and kept dependencies immutable and explicit.
    - **If asked more:** I would explain the three injection types with a preference for constructor injection, and discuss how Spring resolves circular dependencies using three-level cache in singleton scope.
11. Types of dependency injection.
   - **Answer:** There are three types: constructor injection, setter injection, and field injection. In all my production code, I strictly used constructor injection because it enforces immutability and makes testing straightforward. I never used field injection in Talentpace code since it hides dependencies and breaks testability.
    - **If asked more:** I would explain how Spring validates dependencies at startup, why field injection can cause NullPointerException in tests, and how I used `@RequiredArgsConstructor` from Lombok to reduce boilerplate while keeping constructor injection.
12. Constructor injection vs field injection.
   - **Answer:** Constructor injection makes dependencies explicit, immutable, and mandatory. I used constructor injection throughout my CDMS and inventory services. Field injection hides dependencies and makes unit tests harder because you cannot inject mocks through the constructor easily.
    - **If asked more:** I would explain that the Spring team recommends constructor injection, and show how I used constructor injection with Lombok's `@RequiredArgsConstructor` to keep the code clean in all my REST controllers and service classes.
13. What is a Spring bean?
   - **Answer:** A Spring bean is a Java object managed by the Spring IoC container. In my projects, classes annotated with `@Service`, `@Repository`, or `@Component` became beans. For example, my `InventoryValidationService` was a bean with singleton scope, reused across multiple API calls.
    - **If asked more:** I would explain bean naming conventions, how beans are registered via `@ComponentScan` or `@Bean` methods, and how I used `@Scope("prototype")` for a stateful validation context in the inventory engine.
14. What is bean scope?
   - **Answer:** Bean scope determines the lifecycle and visibility of a bean. Singleton scope creates one instance per container, which I used for all my service beans. Prototype creates a new instance every request. In my cold-chain project, I used prototype scope for one-time export DTOs that carried mutable state.
    - **If asked more:** I would explain web-aware scopes like request and session, and how I accidentally discovered singleton scope issues with `@Async` methods — the proxy behavior requires public non-static methods.
15. Difference between singleton and prototype scope.
   - **Answer:** Singleton creates one instance shared across the whole application, which I used for all stateless services like `PartnerService` and `ReportService`. Prototype creates a new instance every time it is injected or requested, which I used sparingly for objects with request-specific state.
    - **If asked more:** I would explain the performance trade-off: singleton saves memory but can have thread-safety issues, while prototype avoids state conflicts but increases GC pressure. I would describe how I resolved a thread-safety issue in a singleton service by removing instance variables.
16. What is application context?
   - **Answer:** ApplicationContext is the Spring IoC container that manages bean lifecycle, event propagation, and internationalization. In my projects, `AnnotationConfigApplicationContext` was created behind the scenes by `SpringApplication.run()`, and I occasionally used `ApplicationContextAware` to access beans programmatically.
    - **If asked more:** I would explain the ApplicationContext hierarchy, how it differs from BeanFactory, and how I used `ConfigurableApplicationContext.close()` in a test `@AfterClass` method to clean up the context.
17. What is bean lifecycle?
   - **Answer:** Bean lifecycle goes through: instantiation, property population, initialization callbacks (`@PostConstruct`, `InitializingBean`), bean is ready, then destruction callbacks (`@PreDestroy`, `DisposableBean`). In my CDMS project, I used `@PostConstruct` to load reference data from MSSQL into a cache after the bean was initialized.
    - **If asked more:** I would explain the full sequence: BeanPostProcessors, `@PostConstruct`, `afterPropertiesSet()`, custom init-method, and how I used `BeanPostProcessor` to log bean initialization times for performance monitoring.
18. What is `@Component`?
   - **Answer:** `@Component` is a stereotype annotation that marks a class as a Spring-managed bean. In my projects, I typically used its specializations like `@Service` and `@Repository` instead of plain `@Component`, since they add semantic meaning and enable persistence exception translation.
    - **If asked more:** I would explain how `@ComponentScan` discovers these annotations, how custom stereotype annotations can be created, and how I used `@Component` for utility classes like `JwtUtil` that didn't fit the service/repository pattern.
19. Difference between `@Component`, `@Service`, `@Repository`, and `@Controller`.
   - **Answer:** All four register Spring beans, but `@Service` is a service layer specialization, `@Repository` enables persistence exception translation, and `@Controller` marks web controllers. In my projects, I used `@Service` for business logic like `InventoryValidationService`, `@Repository` for DAO layers, and `@Controller` for REST endpoints.
    - **If asked more:** I would explain that `@Repository` adds `PersistenceExceptionTranslationPostProcessor` to convert SQLExceptions into Spring's `DataAccessException`, which I relied on in the CDMS project when handling MSSQL constraint violations.
20. What is `@SpringBootApplication`?
   - **Answer:** `@SpringBootApplication` is a convenience annotation combining `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`. Every one of my Talentpace projects had this on the main class, enabling auto-configuration and scanning all beans under the base package.
    - **If asked more:** I would explain how to exclude specific auto-configurations using `exclude` parameter, and how I used `@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})` when setting up a test that didn't need database connectivity.
21. What does `@EnableAutoConfiguration` do?
   - **Answer:** `@EnableAutoConfiguration` enables Spring Boot's auto-configuration mechanism that creates beans based on classpath dependencies. When I added `spring-boot-starter-web`, it automatically configured `DispatcherServlet`, `Jackson ObjectMapper`, and error handling through `ErrorMvcAutoConfiguration`.
    - **If asked more:** I would explain that it's backed by `AutoConfigurationImportSelector` which reads `spring.factories` files, and how I used `spring.autoconfigure.exclude` in `application.properties` when I needed to disable a conflicting auto-configuration for Redis.
22. What is `@Configuration`?
   - **Answer:** `@Configuration` marks a class as a source of bean definitions using `@Bean` methods. In my inventory project, I had a `RedisConfig` class annotated with `@Configuration` where I defined `RedisTemplate` and `CacheManager` beans with custom serialization settings.
    - **If asked more:** I would explain the difference between `@Configuration` with `@Bean` vs `@Component` with `@Autowired`, and how Spring proxies `@Configuration` classes using CGLIB to ensure singleton bean semantics.
23. What is `@Bean`?
   - **Answer:** `@Bean` is a method-level annotation that tells Spring to register the returned object as a bean in the container. In my cold-chain project, I used `@Bean` in a `@Configuration` class to create `KafkaTemplate`, `RedisTemplate`, and `RestTemplate` with project-specific configurations.
    - **If asked more:** I would explain `@Bean` lifecycle callbacks using `initMethod` and `destroyMethod`, and how I configured the `KafkaTemplate` bean with custom serializer properties and retry configuration for the IoT data pipeline.
24. Difference between `@Bean` and `@Component`.
   - **Answer:** `@Bean` is used in `@Configuration` classes for third-party or manually configured beans; `@Component` is for your own classes. I used `@Bean` for `RedisTemplate`, `KafkaTemplate`, and `PasswordEncoder` since these were framework classes, while my services used `@Component` derivatives.
    - **If asked more:** I would explain that `@Bean` gives full control over instantiation and configuration, while `@Component` relies on classpath scanning. I used `@Bean` when I needed to pass constructor arguments like database connection pools or Redis connection factory.
25. What is `application.properties`?
   - **Answer:** `application.properties` is Spring Boot's default configuration file for externalizing settings. In my CDMS project, I stored MSSQL connection URL, username, password, Hibernate DDL strategy, and server port in this file, keeping config separate from code.
    - **If asked more:** I would explain property loading order, profile-specific properties like `application-dev.properties`, and how I used `@Value` and `@ConfigurationProperties` to bind these values to Java objects in my services.
26. Difference between `application.properties` and `application.yml`.
   - **Answer:** Both serve the same purpose but `.properties` is flat key-value while `.yml` uses hierarchical indentation. I preferred `.yml` for my projects because it is more readable for nested configs like `spring.datasource.*` and `spring.jpa.*`, reducing duplication.
    - **If asked more:** I would explain YAML list support and how it improves readability for multi-profile configuration. I'd also mention that YAML does not support `@PropertySource` natively and how I worked around this in the inventory project.
27. What are profiles?
   - **Answer:** Profiles allow environment-specific bean definitions and configurations. I defined `application-dev.yml`, `application-staging.yml`, and `application-prod.yml` for my cold-chain project, each with different database URLs, log levels, and cache settings.
    - **If asked more:** I would explain how `spring.profiles.active` is set via environment variable or JVM argument, and how I used `@Profile("dev")` to conditionally load mock beans for local testing of the Kafka pipeline.
28. How do you externalize configuration?
   - **Answer:** Spring Boot externalizes config through application properties, environment variables, command-line arguments, and `@ConfigurationProperties`. In my inventory project, I kept database credentials in environment variables on the EC2 instance and accessed them via `${DATABASE_URL}` in `application.yml`.
    - **If asked more:** I would explain the `PropertySource` ordering, how I used `@ConfigurationProperties(prefix = "inventory.validation")` to bind nested properties to a POJO, and the benefit of type-safe configuration over `@Value`.
29. What is `@Value`?
   - **Answer:** `@Value` injects a property value from configuration files into a field or parameter. In my CDMS project, I used `@Value("${report.batch-size:1000}")` to configure batch processing size for stored procedure execution, with a sensible default.
    - **If asked more:** I would explain SpEL support in `@Value` for dynamic evaluation, and the trade-off between `@Value` and `@ConfigurationProperties` — `@Value` is simpler but `@ConfigurationProperties` provides type safety and IDE validation.
30. What is `@ConfigurationProperties`?
   - **Answer:** `@ConfigurationProperties` binds entire property hierarchies to strongly-typed Java objects. In my inventory engine, I had an `InventoryProperties` class annotated with `@ConfigurationProperties(prefix = "inventory")` that held validation thresholds, batch sizes, and retry counts.
    - **If asked more:** I would explain `@EnableConfigurationProperties`, nested POJO binding, and how I used `@Validated` with JSR-303 annotations to validate configuration values at startup, preventing production issues from misconfigured properties.
31. What is actuator?
   - **Answer:** Actuator provides production-ready HTTP endpoints for monitoring and managing Spring Boot applications. In my cold-chain project deployed on EC2, I enabled `/health`, `/metrics`, and `/info` endpoints to monitor API health and response times without building custom monitoring endpoints.
    - **If asked more:** I would explain the different endpoint categories, how to expose endpoints via `management.endpoints.web.exposure.include`, and how I extended Actuator by adding a custom `HealthIndicator` that checked Kafka connectivity and MSSQL availability.
32. Which actuator endpoints are useful in production?
   - **Answer:** `/health` for liveness checks, `/metrics` for JVM and request metrics, `/info` for application metadata, and `/loggers` for runtime log level changes. In production, I exposed only `/health` and `/info` publicly and kept others behind the firewall for security.
    - **If asked more:** I would explain how I integrated `/metrics` with Prometheus using `micrometer-registry-prometheus`, and how `/loggers` helped me debug the cold-chain IoT pipeline by enabling DEBUG logging for the Kafka consumer without restarting the JAR.
33. How do you secure actuator endpoints?
   - **Answer:** Actuator endpoints can be secured by restricting exposure, using separate management ports, and applying Spring Security. In my projects, I set `management.endpoints.web.exposure.exclude=*` and only exposed specific endpoints with `include=health,info`, then added `management.server.port=8081`.
    - **If asked more:** I would explain how to use `@RolesAllowed` on custom Actuator endpoints, and how I configured a separate security filter chain for the management port with IP whitelist through a custom `WebSecurityConfigurerAdapter`.
34. What is Spring Boot DevTools?
   - **Answer:** DevTools provides automatic restart, live reload, and remote debugging for development. During local development of my CDMS project, I used DevTools for automatic restart when Java files changed, and I used the LiveReload server to refresh the Swagger UI in the browser.
    - **If asked more:** I would explain how DevTools uses two classloaders, how to exclude resources from restart, and why I never enabled DevTools in production — it can leak sensitive information and causes performance overhead.
35. How do you handle exceptions globally?
   - **Answer:** I handle exceptions globally using `@ControllerAdvice` combined with `@ExceptionHandler` methods. In my CDMS project, I created a `GlobalExceptionHandler` class that caught `DataAccessException`, `MethodArgumentNotValidException`, and custom business exceptions, returning consistent JSON error responses with proper HTTP status codes.
    - **If asked more:** I would explain the exception handling hierarchy, how I mapped `MSSQLException` constraint violations to user-friendly messages, and how I logged stack traces selectively using MDC to include request IDs in logs.
36. What is `@ControllerAdvice`?
   - **Answer:** `@ControllerAdvice` is a global interceptor for controllers that enables cross-cutting exception handling, data binding, and model attributes. In my inventory project, I used a single `@ControllerAdvice` class to handle validation errors, authentication failures, and database constraint violations across all endpoints.
    - **If asked more:** I would explain the difference between `@ControllerAdvice` and `@RestControllerAdvice`, and how I customized the response body with `ErrorResponse` DTOs containing error code, message, timestamp, and trace ID for debugging.
37. What is `@ExceptionHandler`?
   - **Answer:** `@ExceptionHandler` defines a method to handle specific exceptions thrown by controllers. In my global handler, I had methods like `handleValidationException(MethodArgumentNotValidException)` returning 400 with field-level errors, and `handleResourceNotFound(ResourceNotFoundException)` returning 404.
    - **If asked more:** I would explain the priority of exception handlers, how to handle multiple exception types in one method, and how I used `ResponseEntity.exceptionHandler()` for fine-grained control over response headers and status codes in my projects.
38. What is validation in Spring Boot?
   - **Answer:** Validation in Spring Boot uses Bean Validation API (JSR-380) with annotations like `@NotNull`, `@Size`, and `@Pattern` on DTO fields. In my inventory API, I validated incoming serial numbers with `@Pattern(regexp = "^[A-Z0-9]+$")` and checked mandatory fields with `@NotBlank`.
    - **If asked more:** I would explain how validation integrates with `@Valid` in `@RequestBody` parameters, custom validation annotations I created for inventory-specific rules, and how the validation errors are automatically handled by `MethodArgumentNotValidException`.
39. What is `@Valid`?
   - **Answer:** `@Valid` triggers JSR-380 bean validation on request bodies, query parameters, or path variables. In my inventory project, I annotated `@RequestBody InventoryRequest` with `@Valid` in the controller, which automatically validated all field constraints before the service method was called.
    - **If asked more:** I would explain the difference between `@Valid` and `@Validated`, and how `@Validated` supports validation groups — I used validation groups in the CDMS project to have different validation rules for create vs update operations.
40. Difference between `@Valid` and `@Validated`.
   - **Answer:** `@Valid` is standard JSR-380 that triggers validation; `@Validated` is Spring's variant that adds support for validation groups. In my CDMS project, I used `@Validated` with groups like `OnCreate.class` and `OnUpdate.class` to apply different rules for POST and PUT endpoints.
    - **If asked more:** I would explain how to define validation groups using empty interfaces, and how I integrated group validation with `@RequestParam` and `@PathVariable` using `@Validated` at the class level.
41. What is scheduling in Spring Boot?
   - **Answer:** Scheduling in Spring Boot uses `@EnableScheduling` and `@Scheduled` annotations to run tasks periodically. In my CDMS project, I scheduled nightly stored procedure execution at 2 AM using a cron expression to refresh report data without manual intervention.
    - **If asked more:** I would explain the `TaskScheduler` abstraction, how to configure thread pools for scheduled tasks, and the importance of handling failures in scheduled jobs using try-catch blocks to prevent silent task termination.
42. What is `@Scheduled`?
   - **Answer:** `@Scheduled` marks a method to be executed on a schedule. In my inventory project, I used `@Scheduled(fixedDelay = 300000)` on a method that reconciled inventory data every 5 minutes after the previous run completed, ensuring no overlapping executions.
    - **If asked more:** I would explain the three modes: `fixedRate`, `fixedDelay`, and `cron`, and how I used `cron = "0 0 2 * * ?"` in the CDMS project for the nightly ETL batch without needing any external job scheduler initially.
43. Difference between fixed rate and fixed delay.
   - **Answer:** `fixedRate` triggers every N milliseconds regardless of whether the previous execution finished; `fixedDelay` waits N milliseconds after the previous execution completes. In my CDMS project, I used `fixedDelay` for the stored procedure job because overlapping runs would corrupt report data.
    - **If asked more:** I would explain the risk of `fixedRate` causing thread starvation when tasks take longer than the interval, and how I mitigated this in the inventory project by configuring a custom `ThreadPoolTaskScheduler` with a bounded queue.
44. What is cron expression?
   - **Answer:** A cron expression defines schedule using six or seven fields: second, minute, hour, day-of-month, month, day-of-week, and optional year. In my cold-chain project, I used `0 0/15 * * * ?` to run temperature data aggregation every 15 minutes without needing a separate cron job on the server.
    - **If asked more:** I would explain cron syntax with examples, how `?` and `*` differ, and the common mistake of forgetting that cron runs in the server's timezone — I had to adjust the timezone using `zone` attribute in `@Scheduled`.
45. How do you configure scheduled task thread pool?
   - **Answer:** By default, `@Scheduled` uses a single-threaded executor. In my inventory project, I configured a `ThreadPoolTaskScheduler` bean with `pool-size=5` and a custom `ErrorHandler` that logged failures without killing the scheduler, preventing a single failed task from blocking the others.
    - **If asked more:** I would explain how to set a custom `SchedulingConfigurer` with `@Configuration`, and the trade-offs of using a shared thread pool vs dedicated pools for critical vs non-critical scheduled jobs.
46. How do you prevent scheduled jobs from running on all pods?
   - **Answer:** When running multiple instances, scheduled jobs need a coordination mechanism. In my projects deployed on single EC2 instances, this wasn't an issue, but I planned using ShedLock with Redis to ensure only one pod executes a scheduled job at a time by acquiring a distributed lock.
    - **If asked more:** I would explain how ShedLock locks are persisted in a database table or Redis, how to configure lock duration, and how I would integrate ShedLock with `@Scheduled` using `@SchedulerLock(name = "nightlyReport")` for the CDMS batch job.
47. What is async processing?
   - **Answer:** Async processing allows methods to run in a separate thread without blocking the caller. In my cold-chain project, I used async processing for sending email notifications when temperature excursions exceeded thresholds, so the API response wasn't delayed by the email SMTP call.
    - **If asked more:** I would explain the difference between async, reactive, and parallel processing, how async improves API responsiveness, and the threading implications — including the risk of thread pool exhaustion if not configured properly.
48. What is `@Async`?
   - **Answer:** `@Async` marks a method for execution in a separate thread. In my inventory project, I annotated the `AuditLogService.saveAuditLog()` method with `@Async` so that writing audit records to MSSQL wouldn't block the main API response, improving perceived performance.
    - **If asked more:** I would explain that `@Async` requires `@EnableAsync` and only works on public methods called from outside the class (self-invocation bypasses the proxy). I would also explain how I used `CompletableFuture` return types to handle async results.
49. How do you configure async executor?
   - **Answer:** I configure a `ThreadPoolTaskExecutor` bean with custom core pool size, max pool size, queue capacity, and rejection policy. In my cold-chain project, I set `corePoolSize=10`, `maxPoolSize=25`, and `CallerRunsPolicy` to handle spikes in sensor data processing without losing tasks.
    - **If asked more:** I would explain the `AsyncConfigurer` interface, how to handle uncaught exceptions in async methods using `AsyncUncaughtExceptionHandler`, and the impact of queue capacity on memory during traffic bursts in the IoT pipeline.
50. What is caching in Spring Boot?
   - **Answer:** Caching stores frequently accessed data in memory to reduce database load and improve response times. In my cold-chain project, I used Redis cache for storing temperature threshold configurations and partner device mappings, reducing repeated MSSQL queries for data that rarely changed.
    - **If asked more:** I would explain the `@EnableCaching` annotation, cache abstraction layers, cache managers (InMemory vs Redis), and how I measured cache hit ratios in production using Actuator metrics to tune TTL values.
51. What is `@Cacheable`?
   - **Answer:** `@Cacheable` stores the method result in cache and returns it on subsequent calls with the same arguments. I applied `@Cacheable("deviceConfigs")` on the method that fetched gateway-to-device mappings in the cold-chain project, which reduced MSSQL round trips from hundreds per minute to only a few cache misses.
    - **If asked more:** I would explain the cache key generation using `key` attribute and SpEL, conditional caching with `condition` and `unless`, and how I invalidated the cache proactively when device configurations were updated via admin API.
52. Difference between `@Cacheable`, `@CachePut`, and `@CacheEvict`.
   - **Answer:** `@Cacheable` reads and stores; `@CachePut` always executes and updates the cache; `@CacheEvict` removes entries. In my CDMS project, I used `@CachePut` on the partner update method to refresh cached data, and `@CacheEvict(allEntries = true)` on the data reload endpoint to clear stale entries before repopulation.
    - **If asked more:** I would explain `@Caching` for combining multiple cache annotations, and how I used `@CacheEvict(beforeInvocation = true)` to evict cache before method execution when failure should still result in cache being cleared.
53. How do you use Redis cache with Spring Boot?
   - **Answer:** I add `spring-boot-starter-data-redis`, configure Redis connection in `application.yml`, define a `RedisCacheManager` bean, and use `@Cacheable` on service methods. In the cold-chain project, I configured Redis with TTL of 30 minutes for sensor config cache and used `RedisTemplate` for direct operations.
    - **If asked more:** I would explain the difference between `RedisCacheManager` and `RedisTemplate`, how to configure serialization (I used JSON serialization with Jackson2JsonRedisSerializer), and how I handled Redis connection failures by falling back to MSSQL queries.
54. How do you write REST APIs in Spring Boot?
   - **Answer:** I use `@RestController` with `@RequestMapping` for class-level mapping and `@GetMapping`, `@PostMapping`, etc. for HTTP methods. In my CDMS project, I created `PartnerController` with endpoints like `GET /api/partners`, `POST /api/partners/sync`, and `PUT /api/partners/{id}`, returning `ResponseEntity` for status control.
    - **If asked more:** I would explain REST best practices — proper HTTP methods, status codes, request/response DTOs, content negotiation, and how I versioned my APIs using URL path prefix like `/v1/` in the CDMS project.
55. How do you version APIs?
   - **Answer:** I version APIs through the URL path prefix like `/api/v1/partners`. In my inventory project, I maintained backward compatibility by keeping v1 endpoints while adding new fields in v2 request/response DTOs, allowing partners to migrate gradually without breaking their integrations.
    - **If asked more:** I would explain other versioning strategies: header-based (`Accept-version`), query parameter (`?version=1`), and content negotiation. I chose URL path versioning for simplicity since it's explicit in logs and easy to route.
56. How do you document APIs?
   - **Answer:** I document REST APIs using Swagger/OpenAPI 3.0 with `springdoc-openapi` library. In my CDMS project, I added `@Operation` and `@ApiResponse` annotations on controllers to describe endpoints, request bodies, and error responses, making it easy for the frontend team to integrate without constant back-and-forth.
    - **If asked more:** I would explain how I customized the Swagger UI with bearer token support for JWT, how I grouped endpoints by tags, and how I used `springdoc.swagger-ui.enabled=false` in production to expose docs only on staging environments.
57. What is Swagger/OpenAPI?
   - **Answer:** Swagger/OpenAPI is a specification for documenting REST APIs in a machine-readable format (JSON/YAML). In my inventory project, I integrated `springdoc-openapi-starter-webmvc-ui` which auto-generated OpenAPI docs from `@RestController` annotations, and provided an interactive Swagger UI at `/swagger-ui.html`.
    - **If asked more:** I would explain how OpenAPI 3.0 differ from Swagger 2.0, how I defined reusable components (schemas, security schemes) in the OpenAPI config, and how the generated docs helped QA write automated tests using the OpenAPI spec.
58. What is Spring Boot testing?
   - **Answer:** Spring Boot testing uses `@SpringBootTest` for full context integration tests and slice tests for focused layers. In my CDMS project, I wrote integration tests that loaded the full Spring context and tested the REST endpoint from HTTP request to MSSQL persistence using an H2 in-memory database.
    - **If asked more:** I would explain the testing pyramid, how to use `@TestContainers` for testing with real MSSQL, and the importance of `@DirtiesContext` for cleaning up state between tests that modify the application context.
59. What is `@SpringBootTest`?
   - **Answer:** `@SpringBootTest` loads the complete Spring application context for integration testing. In my inventory project, I used `@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)` with `TestRestTemplate` to send real HTTP requests to the API and verify response status, headers, and body.
    - **If asked more:** I would explain how to override properties with `@TestPropertySource` or `properties` attribute, how to use `@MockBean` for external dependencies, and how I configured the test to use an embedded H2 database instead of the production MSSQL.
60. What is `@WebMvcTest`?
   - **Answer:** `@WebMvcTest` loads only the web layer for controller unit testing — it does not load services, repositories, or security. In my CDMS project, I used `@WebMvcTest(PartnerController.class)` with `@MockBean` for the service dependency and tested request mapping, validation, and response serialization.
    - **If asked more:** I would explain the difference between `@WebMvcTest` and `@SpringBootTest`, how I used `MockMvc` for performing requests and assertions, and why `@WebMvcTest` was faster because it avoided scanning all beans in the context.

---

## Spring Security Questions

1. What is Spring Security?
   - **Answer:** Spring Security is a framework for authentication, authorization, and protection against common web vulnerabilities. In my CDMS and inventory projects, I used Spring Security with JWT to secure REST APIs, configured security filter chains, and applied role-based access control for admin and partner users.
   - **If asked more:** I would explain the filter chain architecture, how `SecurityFilterChain` replaced the old `WebSecurityConfigurerAdapter`, and walk through my custom `JwtAuthenticationFilter` that extends `OncePerRequestFilter`.
2. Difference between authentication and authorization.
   - **Answer:** Authentication verifies who you are (identity), authorization verifies what you can do (permissions). In my CDMS project, users logged in with username/password (authentication), and the JWT contained roles like `ROLE_ADMIN` or `ROLE_PARTNER` (authorization) to control access to different API endpoints.
   - **If asked more:** I would explain how authentication creates a `UsernamePasswordAuthenticationToken`, stores it in `SecurityContextHolder`, and how authorization checks `GrantedAuthority` via `@PreAuthorize` or `.hasRole()` in filter chain configuration.
3. What is security filter chain?
   - **Answer:** The security filter chain is a sequence of filters that intercept every HTTP request to apply authentication, authorization, CSRF, CORS, and other security logic. In my inventory project, I configured a custom filter chain with `JwtAuthenticationFilter` before `UsernamePasswordAuthenticationFilter` to validate JWT tokens on every API call.
   - **If asked more:** I would explain how the filter chain is ordered, how `SecurityFilterChain` beans are matched by request matchers, and how I used multiple filter chains for public and private endpoints in the CDMS API.
4. How does Spring Security process a request?
   - **Answer:** Each request passes through the security filter chain. If a JWT is present, my custom filter extracts the token, validates the signature, loads user details, and creates an `Authentication` object stored in `SecurityContextHolder`. If JWT is missing or invalid, the request is rejected with 401 before reaching the controller.
   - **If asked more:** I would walk through the full flow: incoming request → `DelegatingFilterProxy` → `FilterChainProxy` → custom filters → `UsernamePasswordAuthenticationFilter` (for login) → `ExceptionTranslationFilter` → `FilterSecurityInterceptor`.
5. What is `SecurityContextHolder`?
   - **Answer:** `SecurityContextHolder` stores the `SecurityContext` of the currently authenticated user using a `ThreadLocal`. In my CDMS project, I accessed the logged-in user's details in service layers by calling `SecurityContextHolder.getContext().getAuthentication()` to retrieve the username and roles for audit logging.
   - **If asked more:** I would explain the three storage modes: MODE_THREADLOCAL (default), MODE_INHERITABLETHREADLOCAL (for async), and MODE_GLOBAL. I encountered a context loss issue with `@Async` and had to use `SecurityContextHolder.setStrategyName()` to fix it.
6. What is `Authentication` object?
   - **Answer:** `Authentication` represents the authenticated user's identity and authorities. After successful JWT validation in my inventory API, I created a `UsernamePasswordAuthenticationToken` containing the user principal, credentials (null for JWT), and granted authorities, and set it in the `SecurityContextHolder`.
   - **If asked more:** I would explain the three key methods: `getPrincipal()` (user object), `getCredentials()` (password/token), `getAuthorities()` (roles/permissions), and how `isAuthenticated()` flag is set by the `AuthenticationManager`.
7. What is `GrantedAuthority`?
   - **Answer:** `GrantedAuthority` represents a permission granted to the user, like a role or a specific action permission. In my CDMS project, I mapped database roles like `ADMIN`, `PARTNER`, and `VIEWER` to `SimpleGrantedAuthority` objects in the `UserDetails` implementation for authorization checks.
   - **If asked more:** I would explain the difference between role-based (`ROLE_ADMIN`) and permission-based (`REPORT_EXPORT`) authorities, and how I used `hasRole('ADMIN')` vs `hasAuthority('REPORT_EXPORT')` in method security annotations.
8. What is `UserDetails`?
   - **Answer:** `UserDetails` is Spring Security's interface representing the user principal with username, password, authorities, and account status flags. I implemented a custom `UserDetails` class in my inventory project that wrapped the JPA `User` entity and delegated to it for authentication and authorization.
   - **If asked more:** I would explain the methods: `getAuthorities()`, `isAccountNonExpired()`, `isAccountNonLocked()`, `isCredentialsNonExpired()`, and `isEnabled()`, and how I used these for account locking after multiple failed login attempts.
9. What is `UserDetailsService`?
   - **Answer:** `UserDetailsService` is a core interface that loads user-specific data by username. In my CDMS project, I implemented `CustomUserDetailsService` that queried the MSSQL `users` table via JPA repository, mapped the user entity to `UserDetails`, and returned it for authentication.
   - **If asked more:** I would explain the `loadUserByUsername()` method contract, how to handle `UsernameNotFoundException`, and how I cached user details in Redis to avoid querying the database on every request in the inventory project.
10. What is `AuthenticationProvider`?
    - **Answer:** `AuthenticationProvider` processes a specific type of authentication request. In my JWT-based projects, the `DaoAuthenticationProvider` handled username/password login, and I created a custom `JwtAuthenticationProvider` that validated the JWT token and returned the `Authentication` object.
    - **If asked more:** I would explain how to implement a custom `AuthenticationProvider` by overriding `authenticate()` and `supports()`, and how I used `AuthenticationManagerBuilder` to register multiple providers for different authentication types.
11. What is `PasswordEncoder`?
    - **Answer:** `PasswordEncoder` handles password hashing and verification. In all my projects, I used `BCryptPasswordEncoder` for hashing user passwords before storing them in MSSQL, and Spring Security used the same encoder to verify passwords during login authentication.
    - **If asked more:** I would explain the difference between `BCryptPasswordEncoder`, `SCryptPasswordEncoder`, and `Pbkdf2PasswordEncoder`, why BCrypt is preferred for its adaptive salt and work factor, and how I configured the strength parameter in my security config.
12. Why use BCrypt?
    - **Answer:** BCrypt is a slow, salted hashing algorithm designed specifically for passwords. In my CDMS project, I used `BCryptPasswordEncoder` with strength 12 because it makes brute-force attacks impractical — even if the database is compromised, the attacker cannot reverse the hashes easily.
    - **If asked more:** I would explain how BCrypt embeds the salt in the hash output, how the strength factor increases computation time exponentially, and how I chose the strength based on response time requirements (under 1 second per login attempt is acceptable).
13. What is JWT?
    - **Answer:** JWT is a JSON-based token used for stateless authentication consisting of a header, payload, and signature. In my CDMS project, after login the server returned a signed JWT containing the user ID, roles, and expiration time, which the client sent in the `Authorization` header for subsequent requests.
    - **If asked more:** I would explain the three parts of JWT (header with algorithm, payload with claims, signature), how HMAC-SHA256 signing works, and how I used the `io.jsonwebtoken` (jjwt) library to create and parse tokens in my projects.
14. How does JWT authentication work?
    - **Answer:** The client sends credentials to a login endpoint; the server validates them, creates a signed JWT, and returns it. In my inventory API, I had a `POST /auth/login` endpoint that accepted username/password, authenticated via `AuthenticationManager`, generated a JWT with 24-hour expiry, and returned it in the response body.
    - **If asked more:** I would explain the full request flow: client sends `Authorization: Bearer <token>` header → my `JwtAuthenticationFilter` extracts token → validates signature and expiry → parses claims → creates `Authentication` object → sets in `SecurityContextHolder`.
15. What should a JWT contain?
    - **Answer:** A JWT should contain minimal claims: user ID, roles/permissions, issued-at time, and expiration time. In my projects, I included `sub` (user ID), `roles` (comma-separated roles), `iat` (issued at), `exp` (expiration), and `iss` (issuer = application name).
    - **If asked more:** I would explain the difference between standard registered claims (`iss`, `sub`, `aud`, `exp`, `nbf`, `iat`, `jti`) and custom claims, and why I kept the payload small to reduce HTTP header overhead in the cold-chain project with frequent API calls.
16. What should a JWT not contain?
    - **Answer:** A JWT should never contain sensitive information like passwords, credit card numbers, or personally identifiable information (PII) because the payload is base64-encoded, not encrypted. In my projects, I only stored non-sensitive data like user ID and roles — never the password hash or personal details.
    - **If asked more:** I would explain that anyone with the token can decode the payload, and how to use JWE (JWT Encryption) if sensitive claims are needed. I would also mention that token size matters when hitting URL length limits or causing HTTP header overhead.
17. How do you validate JWT signature?
    - **Answer:** The server validates the signature using the same secret key that signed the token. In my CDMS project, I used the jjwt library's `Jwts.parserBuilder().setSigningKey(secretKey).build().parseClaimsJws(token)` which automatically validates the signature, expiry, and throws exceptions for tampered or expired tokens.
    - **If asked more:** I would explain HMAC vs RSA signing, how I stored the secret key in environment variables (not in code), and how I handled edge cases like malformed tokens, expired tokens, and tokens with wrong signature by catching `JwtException` and sending 401 responses.
18. Difference between access token and refresh token.
    - **Answer:** Access tokens are short-lived (15-30 minutes) and used to authenticate API requests; refresh tokens are long-lived (7-30 days) and used to obtain new access tokens without re-login. In my CDMS project, I implemented refresh token rotation where each refresh request invalidated the old refresh token and issued a new pair.
    - **If asked more:** I would explain the security trade-off: longer access tokens increase vulnerability if leaked, while refresh tokens reduce login frequency but need secure storage. I stored refresh tokens in MSSQL with expiry dates for revocation capability.
19. How do you revoke JWT?
    - **Answer:** Stateless JWTs cannot be revoked directly. In my inventory project, I maintained a Redis blacklist of revoked JWT IDs (`jti`) until their natural expiry, and added a filter check at the beginning of the security chain to reject blacklisted tokens.
    - **If asked more:** I would explain alternative approaches: short token expiry reduces revocation window, refresh token revocation invalidates the ability to get new access tokens, and maintaining a deny-list in Redis is the most practical approach with minimal performance impact.
20. What is token rotation?
    - **Answer:** Token rotation issues a new refresh token each time a refresh token is used, invalidating the old one. In my CDMS project, when a client called the `/auth/refresh` endpoint, I validated the current refresh token, revoked it in the database, and issued a new access token and refresh token pair.
    - **If asked more:** I would explain how token rotation limits the damage if a refresh token is stolen — the attacker can use it only once before the legitimate user's next refresh invalidates it. I used `@Transactional` to ensure the rotation was atomic.
21. Where should JWT be stored?
    - **Answer:** On the client side, JWT should be stored in an httpOnly secure cookie or in-memory variable, not in localStorage. In my CDMS project, the frontend React application stored the access token in memory and the refresh token in an httpOnly cookie to mitigate XSS attacks.
    - **If asked more:** I would explain the security implications: localStorage is accessible via JavaScript (XSS vulnerability), while httpOnly cookies are not. I would also discuss the trade-off of in-memory storage where page refresh loses the token, requiring refresh token flow.
22. Why is storing JWT in localStorage risky?
    - **Answer:** localStorage is accessible by any JavaScript running on the same origin, making it vulnerable to XSS attacks. If an attacker injects a script, they can steal the token and impersonate the user. In my projects, I recommended the client team use httpOnly cookies instead.
    - **If asked more:** I would explain CSRF vs XSS risks, how httpOnly cookies prevent XSS but need CSRF protection, and how I configured the backend with CSRF disabled (since my API used Bearer tokens) but advised the frontend team on proper cookie attributes.
23. What is CSRF?
    - **Answer:** CSRF (Cross-Site Request Forgery) tricks an authenticated user into executing unwanted actions on a web application. In my REST API projects, I disabled CSRF protection because JWT-based stateless APIs are not vulnerable to CSRF — there is no session cookie to exploit.
    - **If asked more:** I would explain how CSRF works with session cookies, why stateful form-based apps need CSRF tokens, and how stateless APIs using `Authorization: Bearer` header are immune to CSRF since the attacker's site cannot read the JWT from a different origin.
24. When can CSRF be disabled?
    - **Answer:** CSRF can be disabled when using stateless authentication (JWT), when all state-changing endpoints require a custom header, or when the client and API are on different origins with proper CORS. In my CDMS and inventory projects, I disabled CSRF since I used Bearer tokens exclusively.
    - **If asked more:** I would explain that disabling CSRF without understanding the security implications is dangerous. I would always verify that no session-cookie-based authentication is in use before setting `.csrf().disable()` in the security configuration.
25. What is CORS?
    - **Answer:** CORS (Cross-Origin Resource Sharing) is a browser security mechanism that controls which origins can access a web resource. In my CDMS project, I configured CORS in Spring Security to allow the React frontend hosted on a different domain to make API calls.
    - **If asked more:** I would explain the preflight OPTIONS request, how the `Access-Control-Allow-Origin` header works, and how I configured `allowedOrigins`, `allowedMethods`, and `allowedHeaders` in a `@Bean` CorsConfigurationSource for the inventory project.
26. How do you configure CORS?
    - **Answer:** I configure CORS by defining a `CorsConfigurationSource` bean with allowed origins, methods, and headers. In my cold-chain project, I allowed the React dashboard origin and exposed custom headers like `X-Total-Count` for paginated responses.
    - **If asked more:** I would explain the difference between controller-level `@CrossOrigin` and global CORS configuration, why global config is better for consistency, and how I used `setAllowCredentials(true)` when the frontend needed to send cookies along with JWT in httpOnly cookies.
27. Difference between `hasRole()` and `hasAuthority()`.
    - **Answer:** `hasRole('ADMIN')` automatically prefixes with `ROLE_` to check `ROLE_ADMIN`, while `hasAuthority('REPORT_EXPORT')` checks the exact authority string. In my CDMS project, I used `hasRole('PARTNER')` for partner endpoints and `hasAuthority('REPORT_EXPORT')` for a specific permission that only admins with export permission had.
    - **If asked more:** I would explain how Spring Security adds the `ROLE_` prefix with `roleHierarchy()`, how to customize the prefix, and when to use role-based vs permission-based access control in a multi-tenant inventory system.
28. What is method-level security?
    - **Answer:** Method-level security applies access control at the service or controller method level using `@PreAuthorize`, `@PostAuthorize`, `@Secured`, or `@RolesAllowed`. In my inventory project, I used `@PreAuthorize("hasRole('ADMIN')")` on the partner data reset method so only admin users could trigger it.
    - **If asked more:** I would explain `@EnableMethodSecurity` and the attribute-based expression language — I used SpEL expressions like `@PreAuthorize("hasRole('ADMIN') and #partnerId == authentication.principal.id")` for fine-grained access control.
29. What is `@PreAuthorize`?
    - **Answer:** `@PreAuthorize` evaluates an access control expression before the method executes. In my CDMS project, I used `@PreAuthorize("hasRole('ADMIN')")` on the report generation service to ensure only admin users could generate and download reports, with the security check happening before any business logic.
    - **If asked more:** I would explain the SpEL context variables available — `authentication`, `principal`, and method arguments using `#paramName` — and how I combined multiple conditions with logical operators for complex authorization rules in the inventory service.
30. What is `@PostAuthorize`?
    - **Answer:** `@PostAuthorize` evaluates an access control expression after the method returns, allowing access decisions based on the returned object. In my inventory project, I used `@PostAuthorize("returnObject.partnerId == authentication.principal.partnerId")` to enforce that a user can only view their own partner details.
    - **If asked more:** I would explain the performance considerations — `@PostAuthorize` still executes the method even if access is denied, unlike `@PreAuthorize` which blocks before execution. I would also explain the `returnObject` SpEL variable.
31. What is OAuth2?
    - **Answer:** OAuth2 is an authorization framework where third-party applications get limited access to resources without sharing credentials. In my projects, I didn't use OAuth2 directly — we used JWT with a custom authorization server. But I understand OAuth2 grant types: authorization code, client credentials, and refresh token.
    - **If asked more:** I would explain the roles (resource owner, client, authorization server, resource server), the authorization code flow with PKCE for public clients, and how Spring Security 5 supports OAuth2 resource server configuration with JWT or opaque tokens.
32. Difference between OAuth2 and JWT.
    - **Answer:** OAuth2 is a protocol framework for authorization; JWT is a token format. OAuth2 can use JWT as the token format, but JWT can also be used independently. In my projects, I used JWT directly without OAuth2 — the application both issued and validated tokens for its own APIs.
    - **If asked more:** I would explain that OAuth2 describes how tokens are obtained and refreshed, while JWT defines the token structure and verification mechanism. OAuth2 with JWT introspection is common in microservices, where a gateway validates tokens centrally.
33. What is OpenID Connect?
    - **Answer:** OpenID Connect (OIDC) is an identity layer on top of OAuth2 that adds authentication. It returns an ID token (JWT) containing user identity information. I didn't implement OIDC in my projects, but I understand it solves the authentication gap in OAuth2 by standardizing how the client verifies the user's identity.
    - **If asked more:** I would explain the ID token, access token, and refresh token roles in OIDC, and how `spring-security-oauth2-client` simplifies integration with providers like Google, GitHub, or Azure AD for login.
34. What is resource server?
    - **Answer:** A resource server hosts protected resources and validates access tokens. In my projects, all my Spring Boot APIs acted as resource servers — they received JWTs from clients, validated the signature and claims, and served data only if the token was valid and had sufficient permissions.
    - **If asked more:** I would explain how Spring Security's `oauth2ResourceServer()` DSL configures JWT validation with `jwkSetUri()` or `decoder()`, and how I configured my resource server to use a local signing key for token validation instead of a remote JWKS endpoint.
35. What is authorization server?
    - **Answer:** An authorization server issues access tokens after successful authentication. In my CDMS project, the same Spring Boot application acted as both authorization server (login endpoint) and resource server (API endpoints) — the login endpoint issued JWTs, and the API endpoints validated them.
    - **If asked more:** I would explain that in production microservices, the authorization server should be separate from resource servers. I would discuss Spring Authorization Server, Keycloak, or AWS Cognito as dedicated authorization server solutions.
36. How do you secure actuator endpoints?
    - **Answer:** I secure actuator endpoints by exposing only safe endpoints publicly and applying role-based access. In my inventory project, I exposed `/actuator/health` and `/actuator/info` without authentication, while `/actuator/env` and `/actuator/loggers` required ADMIN role.
    - **If asked more:** I would explain using a dedicated security filter chain for the actuator base path, configuring `management.endpoints.web.exposure.include` carefully, and how I used IP whitelist by adding `hasIpAddress()` constraint in the security rules.
37. How do you handle unauthorized response?
    - **Answer:** I customize the unauthorized response using `AuthenticationEntryPoint` and `AccessDeniedHandler`. In my CDMS project, I implemented a `JwtAuthenticationEntryPoint` that returned a JSON response with 401 status and error message instead of the default HTML login page.
    - **If asked more:** I would show my custom entry point implementation: `response.sendError()` vs writing JSON directly using `response.getWriter().write()`, and how I included a correlation ID in the error response for debugging.
38. Difference between 401 and 403.
    - **Answer:** 401 Unauthorized means the user is not authenticated (no valid JWT). 403 Forbidden means the user is authenticated but lacks permission. In my inventory project, a missing or expired JWT returned 401, while a partner user trying to access admin endpoints returned 403.
    - **If asked more:** I would explain the filter chain flow: `ExceptionTranslationFilter` handles `AuthenticationException` (→401) and `AccessDeniedException` (→403 when authenticated). I used `AccessDeniedHandler` to customize the 403 JSON response with the required role information.
39. How do you implement RBAC?
    - **Answer:** RBAC (Role-Based Access Control) assigns permissions to roles, and roles to users. In my CDMS project, I had roles like `ADMIN`, `PARTNER`, and `VIEWER` stored in MSSQL. The JWT contained the user's role, and `@PreAuthorize("hasRole('ADMIN')")` on sensitive endpoints enforced access.
    - **If asked more:** I would explain the database schema: `users`, `roles`, and `user_roles` tables, how I loaded roles via `UserDetailsService`, and how I used `hasAnyRole()` for endpoints accessible by multiple roles. I would also discuss role hierarchy for inheritance patterns.
40. How do you implement permission-based access?
    - **Answer:** Permission-based access uses granular permissions like `REPORT_CREATE`, `REPORT_EXPORT`, `USER_DELETE` instead of broad roles. In my inventory project, I stored permissions in the `authorities` table and encoded them as `SimpleGrantedAuthority` in the JWT, using `hasAuthority('REPORT_EXPORT')` in security checks.
    - **If asked more:** I would explain the difference between role-based and permission-based access, how to load permissions from the database, and how I created a custom `PermissionEvaluator` for complex permission checks involving entity ownership.
41. How do you secure APIs behind a load balancer?
    - **Answer:** When running behind a load balancer or proxy, Spring Security needs to trust forwarded headers to correctly validate requests. In my EC2-deployed projects behind an ALB, I configured `server.forward-headers-strategy=NATIVE` and used `X-Forwarded-For`, `X-Forwarded-Proto`, and `X-Forwarded-Prefix` headers.
    - **If asked more:** I would explain the `ForwardedHeaderFilter` and `RemoteIpFilter`, how to configure trusted proxies in `application.yml`, and the security implications of misconfigured forwarded headers that could bypass IP-based access controls.
42. How do you handle `Authorization` header through proxies?
    - **Answer:** Proxies and load balancers should preserve the `Authorization` header without modification. In my deployment, we configured the ALB to pass through the `Authorization` header by adding it to the whitelist of forwarded headers, ensuring the JWT reached my Spring Boot application intact.
    - **If asked more:** I would explain that some proxies strip the `Authorization` header for security reasons, and how to use client certificates or API keys as alternatives. I would also discuss configuring `HttpServletRequest` logging to debug header loss in production.
43. What is session fixation?
    - **Answer:** Session fixation is an attack where an attacker forces a user to use a session ID known to the attacker. In my stateless JWT-based APIs, session fixation did not apply since there was no HTTP session. But in stateful apps, Spring Security prevents this by creating a new session on authentication.
    - **If asked more:** I would explain `SessionCreationPolicy.IF_REQUIRED` vs `STATELESS`, how to configure session fixation protection with `sessionManagement().sessionFixation().newSession()`, and why stateless authentication is inherently immune to session fixation.
44. What is stateless session management?
    - **Answer:** Stateless session management means the server does not store any session state between requests. In my CDMS and inventory projects, I configured `SessionCreationPolicy.STATELESS` because I used JWT tokens — the token itself carried all authentication information, eliminating server-side session storage.
    - **If asked more:** I would explain the `SessionCreationPolicy` options (ALWAYS, IF_REQUIRED, NEVER, STATELESS), the benefits of stateless APIs (scalability, no sticky sessions, easier caching), and the trade-off of not being able to invalidate tokens server-side.
45. How do you mix stateful UI and stateless API security?
    - **Answer:** In projects like my cold-chain dashboard, the React UI used session-based authentication for the web pages, and the React app stored a JWT to call the backend APIs. The security configuration used two filter chains: one with session-based auth for the UI routes and one with JWT for the API routes.
    - **If asked more:** I would explain configuring multiple `SecurityFilterChain` beans with `@Order` and `securityMatcher`, how to use `HttpSessionSecurityContextRepository` for stateful parts, and how I managed token refresh in the React app when the JWT expired.

---

## Spring Data JPA and Hibernate Questions

1. What is JPA?
   - **Answer:** JPA (Java Persistence API) is a specification for object-relational mapping in Java. In my CDMS project, I used JPA entities mapped to MSSQL tables with `@Entity`, `@Table`, and `@Column` annotations to interact with the partner and report data without writing manual SQL.
   - **If asked more:** I would explain the difference between a specification (JPA) and an implementation (Hibernate), the key JPA annotations, and how I configured the persistence unit with `spring.jpa.*` properties in `application.yml`.
2. What is Hibernate?
   - **Answer:** Hibernate is the most popular JPA implementation that handles ORM, caching, lazy loading, and transaction management. In my inventory project, Hibernate was the underlying engine for Spring Data JPA — it generated SQL queries, managed the first-level cache, and handled entity state transitions automatically.
   - **If asked more:** I would explain Hibernate-specific features beyond JPA: second-level caching with Redis, Hibernate-specific annotations like `@BatchSize`, `@Fetch`, and HQL query language that differs slightly from JPQL.
3. Difference between JPA and Hibernate.
   - **Answer:** JPA is a specification (interface), Hibernate is an implementation (concrete class). I used JPA annotations and interfaces like `EntityManager` and `@Entity` to keep my code portable, while Hibernate ran underneath. If I ever needed Hibernate-specific features, I used them sparingly to avoid vendor lock-in.
   - **If asked more:** I would explain that Spring Data JPA abstracts both, and I prefer JPA standard APIs for most operations. I would also discuss the differences between Hibernate 5 and 6, and how the `hibernate.jpa.compliance` settings can enforce strict JPA compliance.
4. What is entity?
   - **Answer:** An entity is a Java class mapped to a database table using `@Entity`. In my inventory project, I had `PartnerEntity`, `InventoryRecord`, and `AuditLog` entities with fields annotated with `@Column` to define column names, lengths, and nullability in MSSQL.
   - **If asked more:** I would explain entity requirements: no-arg constructor, `@Id` field, getters/setters, and the importance of proper `equals()` and `hashCode()` using business keys to avoid issues in collections and lazy loading proxies.
5. What is `@Id`?
   - **Answer:** `@Id` marks a field as the primary key of the entity. In my CDMS project, I used `@Id` on the `partnerId` field of `PartnerEntity`, with `@GeneratedValue(strategy = GenerationType.IDENTITY)` to let MSSQL auto-increment the primary key.
   - **If asked more:** I would explain composite primary keys using `@IdClass` or `@EmbeddedId`, and how I used a composite key in the inventory entity to uniquely identify records by `partnerId` and `serialNumber`.
6. What is generated value?
   - **Answer:** `@GeneratedValue` specifies how the primary key is auto-generated. In my projects, I used `GenerationType.IDENTITY` for MSSQL auto-increment columns because it was simpler and more efficient than `SEQUENCE` for my use case.
   - **If asked more:** I would explain the four generation strategies: AUTO, IDENTITY, SEQUENCE, and TABLE, the performance implications of each (IDENTITY disables batch inserts), and why I chose IDENTITY despite the batch insert limitation since my inserts were single-record operations.
7. What is repository?
   - **Answer:** A repository is a Spring Data interface that provides CRUD operations without implementation code. In my inventory project, I created `InventoryRepository extends JpaRepository<InventoryRecord, Long>` and automatically got methods like `findAll()`, `save()`, `deleteById()`, and derived query methods.
   - **If asked more:** I would explain the repository hierarchy: `Repository` → `CrudRepository` → `PagingAndSortingRepository` → `JpaRepository`, and how Spring Data generates the implementation at runtime using `SimpleJpaRepository`.
8. What is Spring Data JPA?
   - **Answer:** Spring Data JPA is a Spring module that reduces JPA boilerplate by providing repository abstractions. In my CDMS project, I defined `PartnerRepository extends JpaRepository` and Spring Data JPA automatically provided implementations for CRUD, pagination, and derived queries without writing any DAO code.
   - **If asked more:** I would explain how Spring Data JPA uses `EntityManager` under the hood, how it translates method names to JPQL queries, and how I used `@Query` annotations for complex MSSQL queries that couldn't be expressed as derived methods.
9. What is `JpaRepository`?
   - **Answer:** `JpaRepository` is a Spring Data interface extending `PagingAndSortingRepository` with JPA-specific methods like `flush()`, `saveAndFlush()`, and `deleteInBatch()`. I used `JpaRepository` for all my MSSQL entities in the inventory project because I needed pagination and batch operations.
   - **If asked more:** I would explain the additional methods `JpaRepository` provides over `CrudRepository`, and when to use `PagingAndSortingRepository` instead of `JpaRepository` for simpler use cases where JPA-specific methods are not needed.
10. Difference between `CrudRepository` and `JpaRepository`.
    - **Answer:** `CrudRepository` provides basic CRUD methods (save, findById, findAll, delete). `JpaRepository` extends it with JPA-specific methods like `flush()`, `saveAndFlush()`, `deleteInBatch()`, and pagination/sorting. I used `JpaRepository` in all my projects since I always needed pagination for the partner listing APIs.
    - **If asked more:** I would explain that `CrudRepository` is persistence-technology agnostic, while `JpaRepository` is JPA-specific. I would also discuss when to use `ListCrudRepository` (Spring Data 3.x) which returns `List` instead of `Iterable`.
11. What is derived query method?
    - **Answer:** Derived query methods allow creating queries by naming methods according to a convention. In my inventory project, I defined `findByPartnerIdAndStatus(Long partnerId, String status)` and Spring Data JPA automatically generated the JPQL query based on the method name — no need to write queries manually.
    - **If asked more:** I would explain the query derivation keywords: `And`, `Or`, `Between`, `Like`, `OrderBy`, `Top`, and how to use `findBy`, `countBy`, `existsBy`, `deleteBy` prefixes. I would also mention the pitfall of too-long method names and when to switch to `@Query`.
12. What is JPQL?
    - **Answer:** JPQL (Java Persistence Query Language) is an object-oriented query language similar to SQL but operates on entities instead of tables. In my CDMS project, I wrote JPQL queries like `SELECT p FROM PartnerEntity p WHERE p.status = :status` to query partners without database-specific SQL.
    - **If asked more:** I would explain the difference between JPQL and SQL — JPQL uses entity names and field names, while SQL uses table and column names. I would also show how I used `@Query("SELECT p FROM PartnerEntity p WHERE p.lastSyncDate < :date")` for custom queries.
13. Difference between JPQL and native query.
    - **Answer:** JPQL is database-independent and works with entity fields; native queries use raw SQL specific to the database. In my CDMS project, I used JPQL for standard queries and native queries with `@Query(value = "EXEC sp_generate_report :partnerId", nativeQuery = true)` for executing MSSQL stored procedures.
    - **If asked more:** I would explain the trade-offs: JPQL is portable but limited for database-specific features, native queries give full control but break database portability. I used native queries only for stored procedures and complex MSSQL window functions.
14. What is entity lifecycle?
    - **Answer:** Entity lifecycle has four states: New (transient), Managed (persistent), Detached, and Removed. In my inventory project, entities retrieved via `findById()` were in the managed state — any changes to them were automatically persisted at flush time without calling `save()`.
    - **If asked more:** I would explain the state transitions: `persist()` moves New → Managed, `merge()` moves Detached → Managed, `remove()` moves Managed → Removed. I encountered detached entity issues when I modified an entity outside a transaction and had to use `merge()`.
15. What is persistence context?
    - **Answer:** The persistence context is a first-level cache that tracks entity state changes within a transaction. In my inventory project, when I loaded a partner entity inside a `@Transactional` service method, the persistence context kept track of all field modifications and flushed them to MSSQL at commit time.
    - **If asked more:** I would explain how the persistence context works as a Map of entity type and `@Id`, how it ensures repeatable reads within a transaction, and how clearing it with `clear()` or `detach()` can help with memory when processing large datasets.
16. What is dirty checking?
    - **Answer:** Dirty checking is Hibernate's mechanism to detect changes to managed entities and automatically persist them. In my CDMS project, I loaded a `PartnerEntity`, updated its `status` field inside a `@Transactional` method, and Hibernate automatically generated an UPDATE query at flush time without me calling `save()`.
    - **If asked more:** I would explain how Hibernate implements dirty checking by comparing snapshots taken at load time with current entity state, and how `hibernate.dirty_checking` configuration can be tuned for performance with large entities.
17. What is first-level cache?
    - **Answer:** The first-level cache is Hibernate's session-level cache within the persistence context. In my inventory project, loading the same entity twice by `findById()` inside the same transaction returned the cached object without a second database query, improving performance for repeated lookups.
    - **If asked more:** I would explain that the first-level cache is always enabled and cannot be disabled, it is scoped to the `EntityManager` (session), and how `clear()` and `evict()` can be used to manage memory for large batch operations.
18. What is second-level cache?
    - **Answer:** The second-level cache is a session-factory-level cache shared across transactions. In my cold-chain project, I configured Hibernate's second-level cache with Redis as the cache store using `hibernate-cache-redis`, caching frequently accessed but rarely changed entity data like gateway configurations.
    - **If asked more:** I would explain the difference between first-level and second-level cache, how to configure second-level cache with `@Cacheable` and `@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)`, and the cache concurrency strategies: READ_ONLY, READ_WRITE, NONSTRICT_READ_WRITE, and TRANSACTIONAL.
19. What is lazy loading?
    - **Answer:** Lazy loading defers loading of associated entities until they are actually accessed. In my CDMS project, the `PartnerEntity` had a `@OneToMany(fetch = FetchType.LAZY)` collection of `OrderEntity` — the orders were loaded from MSSQL only when I accessed `partner.getOrders()`.
    - **If asked more:** I would explain how lazy loading works through Hibernate proxies, the `LazyInitializationException` that occurs when accessing lazy collections outside a transaction, and how I solved it using `@Transactional` or `JOIN FETCH` queries.
20. What is eager loading?
    - **Answer:** Eager loading loads associated entities immediately when the parent entity is fetched. In my inventory project, I used `@ManyToOne(fetch = FetchType.EAGER)` on the `InventoryRecord.partner` field because the partner data was always needed alongside inventory data, avoiding the N+1 problem for that relationship.
    - **If asked more:** I would explain the default fetch types: `@OneToMany` and `@ManyToMany` default to LAZY, while `@ManyToOne` and `@OneToOne` default to EAGER. I would discuss the performance trade-off: eager loading can cause unnecessary joins and Cartesian product issues.
21. What is N+1 query problem?
    - **Answer:** The N+1 problem occurs when 1 query fetches parent entities and N additional queries fetch associated collections. In my CDMS project, `partnerRepository.findAll()` followed by accessing `partner.getOrders()` in a loop triggered N separate queries — making the report generation extremely slow.
    - **If asked more:** I would explain how to detect N+1 by enabling Hibernate SQL logging (`spring.jpa.show-sql=true`), and how I identified the N+1 problem in the CDMS project by seeing hundreds of SELECT queries in the logs for what should have been a single query.
22. How do you solve N+1 query problem?
    - **Answer:** I solve N+1 using `JOIN FETCH` in JPQL or `@EntityGraph`. In my CDMS project, I changed the query from `SELECT p FROM PartnerEntity p` to `SELECT p FROM PartnerEntity p JOIN FETCH p.orders`, which fetched partners and orders in a single SQL query using an INNER JOIN.
    - **If asked more:** I would explain the different solutions: `JOIN FETCH` (eager fetch in query), `@EntityGraph` (declarative), `@BatchSize` (lazy batch loading), and Hibernate's `batch_fetch_size` configuration. I would also discuss the trade-off: JOIN FETCH can cause duplicate results and Cartesian products with multiple collections.
23. What is fetch join?
    - **Answer:** Fetch join is a JPQL feature that loads associations in a single query using JOIN. I used `SELECT p FROM PartnerEntity p JOIN FETCH p.orders` in my inventory project to load partners along with their inventory records in one SQL query, completely avoiding the N+1 problem.
    - **If asked more:** I would explain that fetch join does not change the fetch type of the association but overrides it for that specific query. I would also mention that `LEFT JOIN FETCH` should be used when the association might be null, and that multiple collections require `SET` instead of `LIST` to avoid duplicates.
24. What is entity graph?
    - **Answer:** Entity graphs allow defining fetch plans dynamically using `@NamedEntityGraph` or `@EntityGraph`. In my CDMS project, I defined `@NamedEntityGraph(name = "Partner.withOrders", attributeNodes = @NamedAttributeNode("orders"))` and used `@EntityGraph("Partner.withOrders")` on the repository method for flexible fetch strategies.
    - **If asked more:** I would explain the difference between `EntityGraphType.FETCH` (replaces default fetch plan with only specified attributes) and `EntityGraphType.LOAD` (adds specified attributes to the default plan), and how entity graphs solve N+1 without rewriting JPQL queries.
25. What is pagination?
    - **Answer:** Pagination divides large result sets into smaller pages. In my CDMS project, I used `Pageable` parameter in repository methods: `Page<PartnerEntity> findAll(Pageable pageable)`. The API returned page number, size, total elements, and total pages, allowing the React frontend to display paginated tables.
    - **If asked more:** I would explain `PageRequest.of(page, size, Sort)`, how Spring Data JPA translates pagination to database-specific SQL (using `OFFSET` and `FETCH NEXT` in MSSQL), and the performance downside of large offsets — I switched to keyset pagination for very large datasets.
26. What is sorting?
    - **Answer:** Sorting orders query results by specified fields. In my inventory project, I used `Sort.by("partnerName").ascending()` in the repository method or passed `Sort` through `Pageable`. For multi-field sorting, I used `Sort.by(Order.asc("partnerName"), Order.desc("createdDate"))`.
    - **If asked more:** I would explain the difference between dynamic sorting via `Pageable` parameter (user-controlled) and static sorting via `@OrderBy` annotation on entity associations. I would also mention the SQL injection risk of field sorting with `Sort.by()` — Spring validates it, but custom implementations need care.
27. What is transaction management?
    - **Answer:** Transaction management ensures a group of database operations either all succeed or all fail. In my inventory project, I used Spring's declarative transaction management with `@Transactional` on service methods to ensure atomicity — if one validation step failed, the entire batch update was rolled back.
    - **If asked more:** I would explain ACID properties, the difference between programmatic (TransactionTemplate) and declarative (@Transactional) transaction management, and how Spring wraps the service in a proxy to handle begin, commit, and rollback automatically.
28. What is `@Transactional`?
    - **Answer:** `@Transactional` ensures a set of database operations either all succeed or all roll back. In my inventory project, I used it on the batch validation service so that if any partner record failed validation, the entire batch was rolled back to maintain data consistency.
    - **If asked more:** I would explain the attributes: `propagation`, `isolation`, `timeout`, `readOnly`, and `rollbackFor`. I set `readOnly = true` on read-only methods for performance optimization and `rollbackFor = CustomException.class` to specify which exceptions trigger rollback.
29. How does `@Transactional` work internally?
    - **Answer:** Spring uses AOP proxies to wrap `@Transactional` methods. When a method annotated with `@Transactional` is called, Spring creates a transaction before method execution and commits or rolls back based on the outcome. In my projects, I confirmed this works via CGLIB proxy — which means self-invocation bypasses the proxy and the annotation is ignored.
    - **If asked more:** I would explain the `TransactionInterceptor` and `PlatformTransactionManager` chain, how `TransactionAspectSupport` manages transaction status, and the difference between JDBC and JPA transaction managers — I used `JpaTransactionManager` for my MSSQL entities.
30. What is transaction propagation?
    - **Answer:** Transaction propagation defines how transactions behave when a transactional method calls another transactional method. In my CDMS project, I used `REQUIRED` (default) for most methods, meaning they join the existing transaction. For the audit logging method, I used `REQUIRES_NEW` so the audit log was saved even if the main transaction rolled back.
    - **If asked more:** I would explain the seven propagation types: REQUIRED, SUPPORTS, MANDATORY, REQUIRES_NEW, NOT_SUPPORTED, NEVER, and NESTED. I used REQUIRES_NEW in the inventory project for the notification service to send alerts independently of the main transaction outcome.
31. What is transaction isolation?
    - **Answer:** Transaction isolation defines how concurrent transactions interact. In my inventory project, I used `@Transactional(isolation = Isolation.READ_COMMITTED)` which is MSSQL's default and prevents dirty reads. For the inventory calculation method, I considered `REPEATABLE_READ` to prevent phantom reads during stock computation.
    - **If asked more:** I would explain the four isolation levels: READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, and SERIALIZABLE, and the problems they prevent: dirty read, non-repeatable read, and phantom read. I would also discuss the performance trade-off of higher isolation levels.
32. What is rollback behavior?
    - **Answer:** By default, `@Transactional` rolls back on unchecked exceptions (RuntimeException) and commits on checked exceptions. In my projects, I customized this with `rollbackFor = {DataAccessException.class, CustomValidationException.class}` to ensure certain checked exceptions also triggered rollback.
    - **If asked more:** I would explain `noRollbackFor` for cases where rollback should not happen, and how I configured declarative rollback rules in XML or Java configuration for finer control over which exceptions roll back or commit the transaction.
33. Why does self-invocation break `@Transactional`?
    - **Answer:** Self-invocation bypasses the Spring AOP proxy because the call happens within the same class, not through the injected proxy. In my inventory project, I accidentally faced this when method A (annotated with `@Transactional`) called method B (also `@Transactional`) in the same service — the transaction for B was ignored.
    - **If asked more:** I would explain the proxy pattern and how Spring creates a CGLIB proxy (or JDK proxy for interfaces). Solutions I used: self-injecting the proxy via `ApplicationContextAware` or `@Lazy`, extracting transactional methods into a separate service, or using `TransactionTemplate` programmatically.
34. What is optimistic locking?
    - **Answer:** Optimistic locking assumes conflicts are rare and checks for them at commit time using a version column. In my inventory project, I added `@Version` on the `InventoryRecord` entity to prevent concurrent updates — if two users tried to update the same record, the second commit threw `OptimisticLockException`.
    - **If asked more:** I would explain how optimistic locking works without database locks (just a version check), the `StaleObjectStateException`, and how I handled the exception in the service layer by retrying the operation or informing the user that the data was modified by someone else.
35. What is pessimistic locking?
    - **Answer:** Pessimistic locking locks the database row at read time to prevent other transactions from modifying it. In my CDMS project, I used `@Lock(LockModeType.PESSIMISTIC_WRITE)` on the repository method that calculated partner inventory, ensuring no other transaction could modify the data during the calculation.
    - **If asked more:** I would explain the lock types: PESSIMISTIC_READ (shared lock), PESSIMISTIC_WRITE (exclusive lock), and how they translate to `SELECT ... FOR UPDATE` in MSSQL. I would discuss the trade-off: pessimistic locking prevents conflicts but reduces concurrency compared to optimistic locking.
36. What is `@Version`?
    - **Answer:** `@Version` is a JPA annotation that enables optimistic locking by maintaining a version number. In my inventory project, I added `@Version private Long version;` to the `InventoryRecord` entity. Hibernate automatically incremented the version on every update and checked it before committing, throwing `OptimisticLockException` on conflict.
    - **If asked more:** I would explain that `@Version` works with any numeric type, `Timestamp`, or `Instant`. I would also discuss the common mistake of forgetting to include `@Version` and the result — silent data overwrites in concurrent scenarios.
37. What are cascade types?
    - **Answer:** Cascade types determine which entity operations propagate to associated entities. In my CDMS project, I used `cascade = CascadeType.PERSIST` on the `@OneToMany(mappedBy = "partner")` orders collection so that saving a partner also saved its orders without separate save calls.
    - **If asked more:** I would explain the cascade types: ALL, PERSIST, MERGE, REMOVE, REFRESH, DETACH, and the common mistake of using `CascadeType.ALL` on `@ManyToMany` which causes unintended deletes. I used only `PERSIST` and `MERGE` in my projects to avoid accidental removals.
38. What is orphan removal?
    - **Answer:** Orphan removal automatically deletes child entities when they are removed from the parent's collection. In my inventory project, I used `orphanRemoval = true` on the `@OneToMany` partner-orders mapping so that when an order was removed from the partner's order list, it was automatically deleted from MSSQL.
    - **If asked more:** I would explain the difference between `orphanRemoval = true` and `cascade = CascadeType.REMOVE` — orphan removal removes children removed from the collection, while cascade REMOVE deletes children when the parent is deleted. I used both for different scenarios.
39. Difference between `save()` and `saveAndFlush()`.
    - **Answer:** `save()` persists the entity and returns it, but does not immediately flush to the database — the actual INSERT/UPDATE happens at transaction commit. `saveAndFlush()` immediately flushes the SQL to the database. In my inventory project, I used `saveAndFlush()` when I needed the generated ID immediately for logging purposes.
    - **If asked more:** I would explain that `save()` may batch multiple operations before flushing, improving performance. `saveAndFlush()` is useful when the next operation depends on the entity being persisted (like calling a stored procedure that needs the entity's ID).
40. What is batch insert?
    - **Answer:** Batch insert groups multiple INSERT statements into a single database round-trip for performance. In my CDMS project, I configured `spring.jpa.properties.hibernate.jdbc.batch_size=50` and `hibernate.order_inserts=true` so that inserting 500 partner records in a loop triggered only 10 batch inserts instead of 500 individual queries.
    - **If asked more:** I would explain the requirements for batch inserts: IDENTITY generator disables batch inserts (so I used SEQUENCE or TABLE generator), `order_inserts` and `order_updates` must be enabled, and `rewriteBatchedStatements=true` for MSSQL. I learned this when testing batch inserts for the inventory import feature.
41. How do you improve JPA performance?
    - **Answer:** I improve JPA performance by enabling batch operations, using fetch joins to avoid N+1 queries, configuring appropriate fetch types, enabling Hibernate query cache for read-heavy data, and monitoring slow queries via `spring.jpa.properties.hibernate.generate_statistics=true`.
    - **If asked more:** I would explain specific optimizations I applied: setting `batch_size=50` for bulk inserts, using `@BatchSize` on lazy collections, avoiding `select *` by using projection DTOs, and switching to native SQL or stored procedures for complex reporting queries that couldn't be optimized through JPA.
42. When should you use JDBC instead of JPA?
    - **Answer:** Use JDBC when performance is critical for bulk operations, complex reporting queries, or when executing stored procedures. In my CDMS project, I used `JdbcTemplate` for the stored procedure execution because it was simpler and faster than mapping complex result sets to entities for the reporting use case.
    - **If asked more:** I would explain that JPA overhead (persistence context, dirty checking, lazy loading) becomes significant with thousands of entities in memory. For the CDMS nightly batch that processed millions of records, JPA would have caused memory issues — JDBC was the right choice.
43. When should you use stored procedures?
    - **Answer:** Use stored procedures for complex reporting logic, long-running batch operations, or when you need database-specific features that cannot be expressed in JPA. In my CDMS project, the nightly report generation was implemented as an MSSQL stored procedure because it involved complex joins, aggregations, and conditional logic that was more efficient in SQL.
    - **If asked more:** I would explain the benefits: reduced network round-trips (logic runs in the database), better performance for set-based operations, and the ability to use MSSQL-specific features like window functions. The trade-off is testing complexity and database portability.
44. How do you call stored procedures from Spring?
    - **Answer:** I call stored procedures using `@Procedure` annotation on a repository method, or `JdbcTemplate` for direct execution. In my CDMS project, I used `@Procedure(procedureName = "sp_generate_report")` on a method in my JPA repository and mapped the result to a DTO using a custom `ResultSetExtractor`.
    - **If asked more:** I would explain the three approaches: `@Procedure` annotation (simplest, but limited to single result sets), `EntityManager.createStoredProcedureQuery()` (more control), and `JdbcTemplate` (full control). I used `JdbcTemplate` for complex stored procedures that returned multiple result sets or had output parameters.
45. How do you debug slow JPA queries?
    - **Answer:** I enable Hibernate SQL logging with `spring.jpa.show-sql=true` and `spring.jpa.properties.hibernate.format_sql=true`, and statistics logging with `hibernate.generate_statistics=true`. In my CDMS project, this helped me identify N+1 queries and missing indexes by analyzing the generated SQL in the logs.
    - **If asked more:** I would explain how I used `spring.jpa.properties.hibernate.use_sql_comments=true` to trace which service method generated each query, and how I copied slow queries from logs to MSSQL Management Studio to analyze execution plans and identify missing indexes or expensive joins.

---

## Database and SQL Questions

1. What is DBMS?
   - **Answer:** A DBMS is software that stores, manages, and retrieves data systematically. In CDMS, we used MSSQL as our DBMS to handle Lenovo's channel partner data, and I worked heavily with execution plans and indexing to keep queries fast.
   - **If asked more:** I would explain how I analyzed wait statistics in MSSQL to identify where the DBMS was spending most of its time during the 5-hour stored procedure runs.
2. What is RDBMS?
   - **Answer:** An RDBMS stores data in related tables using keys and indexes. MSSQL is the RDBMS I used at Talentpace, where I designed normalized schemas for inventory data with foreign keys linking partners, products, and transactions.
   - **If asked more:** I would contrast RDBMS with NoSQL and explain why we chose MSSQL — strong ACID compliance was critical for inventory accuracy and reporting.
3. What is primary key?
   - **Answer:** A primary key uniquely identifies each row in a table. In the inventory system, I used `SerialNumber` as the primary key to ensure every serial record was unique and could be traced back to a specific partner shipment.
   - **If asked more:** I would discuss how primary key choice affects clustered index performance, and why I prefer narrow integer keys over wide composite keys in high-volume tables.
4. What is foreign key?
   - **Answer:** A foreign key links two tables and enforces referential integrity. In CDMS, I used foreign keys to ensure each report record pointed to a valid partner ID, preventing orphan data when partners were deactivated.
   - **If asked more:** I would explain how missing foreign key indexes caused nested loop joins in the 5-hour procedure, and adding them was one of my first optimization steps.
5. What is unique key?
   - **Answer:** A unique key ensures all values in a column or column set are distinct. In the inventory validation engine, I applied a unique constraint on (PartnerID, SerialNumber) to prevent duplicate serial records from entering the system.
   - **If asked more:** I would explain the difference between unique constraint and unique index in MSSQL, and how I used filtered unique constraints for active records only.
6. Difference between primary key and unique key.
   - **Answer:** A table has only one primary key, which is clustered by default in MSSQL, while you can have multiple unique keys. I used primary keys for entity identity and unique keys for business-level uniqueness like partner codes.
   - **If asked more:** I would talk about how choosing the wrong clustered index (on a wide unique key instead of a narrow primary key) degraded insert performance in the inventory batch load.
7. What is index?
   - **Answer:** An index is a database structure that speeds up data retrieval. In CDMS, I created composite indexes on columns used in WHERE and JOIN conditions of the slow stored procedures, which changed table scans to index seeks.
   - **If asked more:** I would explain how I used the Database Engine Tuning Advisor recommendations alongside manual analysis to decide which indexes to create.
8. How does indexing improve performance?
   - **Answer:** Indexing creates a sorted data structure (B-tree in MSSQL) that allows the engine to locate rows without scanning the entire table. I saw this firsthand when adding a covering index reduced a 45-minute query in CDMS to under 30 seconds.
   - **If asked more:** I would explain how the query optimizer chooses between index seek, scan, and lookup based on index structure and statistics.
9. What are the disadvantages of indexes?
   - **Answer:** Indexes slow down INSERT, UPDATE, and DELETE operations because the index must be maintained. In the inventory system, I had to balance read performance for validation queries against the write overhead during the batch ingestion of 10,000+ serial records.
   - **If asked more:** I would discuss how I monitored index fragmentation and set up weekly rebuild jobs to prevent performance degradation over time.
10. What is clustered index?
    - **Answer:** A clustered index determines the physical order of data in a table. In MSSQL, the primary key creates a clustered index by default. In CDMS, I chose the report date as the clustered index key for a large fact table to optimize range-based reporting queries.
    - **If asked more:** I would warn about choosing clustered indexes on incrementing columns to avoid page splits, which I observed during high-volume inserts in the inventory pipeline.
11. What is non-clustered index?
    - **Answer:** A non-clustered index is a separate structure that contains key values and pointers to the actual rows. I created non-clustered indexes on foreign key columns in CDMS to speed up JOIN operations between the partner and transaction tables.
    - **If asked more:** I would explain how include columns in non-clustered indexes can make them covering indexes, which I used extensively to eliminate key lookups.
12. What is composite index?
    - **Answer:** A composite index is an index on multiple columns, with the column order determining its effectiveness. In CDMS, I created a composite index on (PartnerID, ReportDate) because the slow procedure always filtered by partner first, then by date range.
    - **If asked more:** I would explain how column order matters — leading with the most selective column gives the best performance, which I verified by comparing estimated execution plans.
13. What is covering index?
    - **Answer:** A covering index includes all columns referenced by a query, eliminating the need to access the table data. In CDMS, I rebuilt several non-clustered indexes with INCLUDE columns for the SELECT list, which eliminated expensive key lookups.
    - **If asked more:** I would discuss the trade-off — covering indexes are larger and increase maintenance cost, so I only created them for the most frequent queries.
14. What is filtered index?
    - **Answer:** A filtered index indexes only a subset of rows based on a WHERE condition. I used a filtered index on the inventory table where `Status = 'Active'` to keep the index small and fast for active-record lookups.
    - **If asked more:** I would explain how filtered indexes also maintain better statistics for the relevant data subset, which helped the optimizer choose better plans.
15. What is index selectivity?
    - **Answer:** Selectivity measures how many rows match a given value — high selectivity means few rows per value. In CDMS, I analyzed column selectivity to decide index order: PartnerID had high selectivity, so I put it first in composite indexes.
    - **If asked more:** I would explain how low-selectivity indexes (like on a boolean flag) are often ignored by the optimizer, which I confirmed through actual execution plan analysis.
16. What is query execution plan?
    - **Answer:** An execution plan shows how the database engine processes a query — which indexes it uses, how it joins tables, and where the cost is. In the CDMS optimization, I spent the first week reading actual execution plans for each subquery in the stored procedure.
    - **If asked more:** I would walk through how I read an actual plan: looking for table scans, hash joins with large row estimates, and operators with high estimated vs actual row counts indicating stale statistics.
17. How do you analyze an execution plan?
    - **Answer:** I start with the most expensive operator (highest % of cost), check if it's a scan vs seek, look for missing index suggestions, and compare estimated vs actual rows. In CDMS, I found a Cartesian product because a join predicate was missing — it showed as a massive nested loop with 100M+ rows.
    - **If asked more:** I would explain how I enable actual execution plans in SSMS, use SET STATISTICS TIME/IO ON for detailed metrics, and compare plans before and after changes.
18. What is table scan?
    - **Answer:** A table scan reads every row in the table to find matching data. In CDMS, I found table scans on a 5-million-row transaction table because there was no index on the join column — this was the primary reason the report took 5 hours.
    - **If asked more:** I would explain that table scans are acceptable for small tables but disastrous for large ones, and how adding a single index converted the scan to a seek.
19. What is index seek?
    - **Answer:** An index seek navigates the B-tree to find only the relevant rows, making it highly efficient. After I added the composite index on (PartnerID, ReportDate) in CDMS, the execution plan changed from a table scan to an index seek, dropping query time from minutes to seconds.
    - **If asked more:** I would explain how seeks depend on sargable WHERE conditions, and how I rewrote non-sargable functions like `YEAR(DateCol)` to range comparisons.
20. What is index scan?
    - **Answer:** An index scan reads all rows in an index (non-clustered) rather than the table, which is faster than a table scan but slower than a seek. In the inventory system, I saw index scans on a date index when queries didn't filter selectively enough.
    - **If asked more:** I would explain the difference between table scan and index scan — index scan reads fewer pages since the index is narrower, but it's still expensive for large tables.
21. What is key lookup?
    - **Answer:** A key lookup happens when a non-clustered index doesn't cover the query, so the engine fetches the actual row from the clustered index. In CDMS, I eliminated key lookups by adding INCLUDE columns to existing non-clustered indexes.
    - **If asked more:** I would explain how multiple key lookups per row compound into thousands of random I/O operations, and how I used the index tuning advisor to identify them.
22. What are joins?
    - **Answer:** Joins combine rows from two or more tables based on related columns. In CDMS, I optimized joins by ensuring both sides had proper indexes and that join predicates used the correct columns — one missing predicate had caused a cross join between a 500K and 200K row table.
    - **If asked more:** I would explain the three physical join operators (nested loop, hash match, merge join) and how I influenced the optimizer's choice through indexing and query hints.
23. Difference between inner join and left join.
    - **Answer:** Inner join returns only matching rows from both tables; left join returns all rows from the left table and matching from the right. In the inventory reconciliation, I used left joins to find serial records that existed in the source but not in the target.
    - **If asked more:** I would explain how accidentally using a left join where an inner join sufficed caused incorrect row counts in CDMS reports, which I caught by comparing before-and-after output.
24. Difference between left join and right join.
    - **Answer:** Left join preserves all left-table rows; right join preserves all right-table rows. I almost always use left joins for readability and consistency, since right joins can make query logic harder to follow.
    - **If asked more:** I would share a real debugging story where a right join inside a CTE caused confusing results in CDMS, and I refactored it to left joins for clarity.
25. Difference between full join and cross join.
    - **Answer:** Full join returns all rows from both tables with matches where available; cross join returns the Cartesian product of both tables. In a CDMS data validation script, I accidentally wrote a cross join by omitting the ON clause, which produced 50M rows instead of 5K.
    - **If asked more:** I would explain how I now always inspect actual execution plans to catch accidental Cartesian products, since the optimizer shows them clearly.
26. What is normalization?
    - **Answer:** Normalization reduces data redundancy by splitting tables into related entities. In the inventory system, I normalized partner data into separate tables (Partners, Products, Transactions) to avoid duplicate storage and update anomalies.
    - **If asked more:** I would discuss how normalization helped data integrity but required careful indexing on foreign keys to maintain join performance.
27. What are normal forms?
    - **Answer:** Normal forms are progressive rules to eliminate redundancy: 1NF removes repeating groups, 2NF removes partial dependencies, 3NF removes transitive dependencies. Our CDMS schema was in 3NF, which I verified before designing indexes.
    - **If asked more:** I would explain how I sometimes denormalized specific report tables for performance while keeping the transactional schema in 3NF.
28. What is denormalization?
    - **Answer:** Denormalization intentionally adds redundancy for read performance. In CDMS, I added computed columns for frequently calculated metrics so the stored procedures didn't have to compute them on the fly across millions of rows.
    - **If asked more:** I would explain the trade-off: denormalization speeds up reads but complicates writes, so I only applied it to reporting tables, not transactional ones.
29. When should you denormalize?
    - **Answer:** Denormalize when read performance is critical and the data is mostly static, like pre-aggregated reporting tables. In CDMS, I denormalized monthly partner summaries into a separate table that the dashboard queried instead of aggregating raw data each time.
    - **If asked more:** I would explain how I used indexed views in MSSQL as a middle ground — they look like views but are physically stored and maintained by the engine.
30. What is stored procedure?
    - **Answer:** A stored procedure is a pre-compiled batch of SQL statements stored in the database. The CDMS reporting system relied entirely on stored procedures, and I optimized the main one from 5 hours to 12 minutes by rewriting queries and fixing indexes.
    - **If asked more:** I would explain how I profiled each section of the procedure using SET STATISTICS TIME and broke it down to find the individual expensive queries.
31. Advantages and disadvantages of stored procedures.
    - **Answer:** Advantages: pre-compiled execution plans, reduced network traffic, centralized business logic. Disadvantages: harder to version control, debug, and test compared to application code. In CDMS, I felt this pain when I couldn't easily diff procedure versions.
    - **If asked more:** I would explain my approach: keep reporting logic in procedures (where set-based operations are fast) and business validation in application code (where it's testable).
32. What is function in SQL?
    - **Answer:** A function returns a scalar value or table. In CDMS, I used scalar functions for reusable logic like date formatting, but I later replaced them with inline table-valued functions because scalar functions caused row-by-row execution.
    - **If asked more:** I would explain how scalar functions in WHERE clauses made queries non-sargable, and how converting them to inline TVFs eliminated the performance issue.
33. Difference between stored procedure and function.
    - **Answer:** Functions must return a value and cannot modify data; stored procedures can modify data and have side effects. I use stored procedures for ETL operations and functions for computed columns or reusable lookups.
    - **If asked more:** I would explain that functions in FROM clauses can be inline (optimized like views) or multi-statement (materialized temp tables), and I prefer inline for performance.
34. What is trigger?
    - **Answer:** A trigger runs automatically on INSERT, UPDATE, or DELETE. In the inventory system, I used an AFTER INSERT trigger on the serial records table to update the partner's last-activity timestamp automatically.
    - **If asked more:** I would warn against complex triggers that cause cascading issues — I once debugged a trigger chain that locked the inventory table for minutes during bulk inserts.
35. What is view?
    - **Answer:** A view is a saved SQL query that looks like a table. In CDMS, I created views for frequently joined table combinations so report developers didn't have to remember complex join conditions.
    - **If asked more:** I would explain how views can hide complexity but also hide performance problems — a view joining 10 tables looks simple but runs like 10 tables.
36. What is materialized view?
    - **Answer:** In MSSQL, indexed views persist the result set physically, updated automatically. I used an indexed view for the monthly partner sales summary in CDMS, which kept the dashboard queries fast without manual ETL.
    - **If asked more:** I would explain the restrictions on indexed views (must use SCHEMABINDING, no DISTINCT in aggregates) and how I worked around them by using COUNT_BIG instead of COUNT.
37. What is transaction?
    - **Answer:** A transaction groups multiple operations into an atomic unit — all succeed or all roll back. In the inventory validation pipeline, I wrapped the batch insert of 10,000 serial records in a transaction to ensure partial failures didn't corrupt the inventory state.
    - **If asked more:** I would explain how I chose transaction scope — keeping it short to avoid holding locks, and how I handled deadlocks when concurrent batches ran.
38. What are ACID properties?
    - **Answer:** Atomicity (all-or-nothing), Consistency (valid state before and after), Isolation (concurrent transactions don't interfere), Durability (committed data survives failures). MSSQL's ACID compliance was why we chose it for inventory — we couldn't lose or corrupt serial-level data.
    - **If asked more:** I would explain how isolation levels trade strictness for performance, and how I used snapshot isolation in CDMS for read-only reporting to avoid writer blocking.
39. What is isolation level?
    - **Answer:** Isolation level controls how transactions see each other's changes. In CDMS reporting, I used READ UNCOMMITTED (with NOLOCK hints) on the read-only report queries because slight dirty reads were acceptable and we needed zero blocking on writes.
    - **If asked more:** I would explain the spectrum from READ UNCOMMITTED (dirty reads possible) to SERIALIZABLE (no concurrency), and how I chose based on whether accuracy or speed mattered.
40. Difference between read committed and repeatable read.
    - **Answer:** Read committed prevents dirty reads but allows non-repeatable reads; repeatable read prevents both. In inventory reconciliation, I used repeatable read to ensure two successive reads of the same baseline data matched during validation.
    - **If asked more:** I would explain how repeatable read holds shared locks until the transaction ends, which increased blocking but was necessary for reconciliation accuracy.
41. What is dirty read?
    - **Answer:** A dirty read occurs when a transaction reads uncommitted data from another transaction. I allowed dirty reads in the CDMS dashboard (via NOLOCK hints) because stale data by a few seconds was acceptable for monitoring, but never in the inventory validation engine.
    - **If asked more:** I would explain how NOLOCK can also cause missed or double-read rows due to page splits, which is why I avoided it for any financial or count-based reports.
42. What is non-repeatable read?
    - **Answer:** A non-repeatable read happens when a row changes between two reads in the same transaction. In the inventory pipeline, I encountered this when the baseline data was being updated by another process during validation, so I switched to snapshot isolation.
    - **If asked more:** I would compare how different isolation levels handle this, and why snapshot isolation gives consistency without blocking.
43. What is phantom read?
    - **Answer:** A phantom read occurs when new rows appear between two reads in the same transaction. In CDMS, phantom reads didn't affect our aggregate reports significantly, so we stayed at READ COMMITTED for most reporting.
    - **If asked more:** I would explain how SERIALIZABLE or snapshot isolation prevents phantoms by locking ranges or using row versioning.
44. What is deadlock?
    - **Answer:** A deadlock occurs when two transactions each hold a lock the other needs, and SQL Server chooses a victim. In the inventory system, concurrent batch ingestion processes occasionally deadlocked on the serial records table, and one process was killed automatically.
    - **If asked more:** I would explain how I reduced deadlocks by ensuring all transactions accessed tables in the same order and kept transaction durations short.
45. How do you prevent deadlocks?
    - **Answer:** I prevent deadlocks by accessing tables in a consistent order across transactions, keeping transactions short, and using appropriate isolation levels. In the inventory pipeline, I also used row versioning to reduce lock contention.
    - **If asked more:** I would explain how I used SQL Server Profiler to capture deadlock graphs and visualized them to understand which queries and objects were involved.
46. What is locking?
    - **Answer:** Locking controls concurrent access to data. MSSQL uses row, page, and table locks based on the operation. In CDMS, I saw lock escalation from row to table locks during bulk report generation, which blocked other queries until I broke the batch into smaller chunks.
    - **If asked more:** I would explain the lock hierarchy and how lock escalation works, and how I monitored locks with `sys.dm_tran_locks`.
47. What is row lock?
    - **Answer:** A row lock locks a single row to allow maximum concurrency. In the inventory system, row locks allowed multiple partner feeds to be processed concurrently on different serial records without blocking each other.
    - **If asked more:** I would explain how row locks consume memory and can escalate, and how I checked lock granularity using execution plans.
48. What is table lock?
    - **Answer:** A table lock locks the entire table, preventing any concurrent access. In CDMS, the original stored procedure escalated to table locks on the transactions table during the long-running report, blocking all incoming data feeds.
    - **If asked more:** I would explain how I used `ALTER TABLE ... SET LOCK_ESCALATION = AUTO` and partitioned large tables to limit lock escalation scope.
49. What is optimistic locking?
    - **Answer:** Optimistic locking assumes conflicts are rare and checks at commit time using a version column. I used a rowversion column in the inventory entity to detect concurrent modifications during reconciliation without holding long locks.
    - **If asked more:** I would explain how I handled the `DbUpdateConcurrencyException` in the application layer by retrying the operation after refreshing the data.
50. What is pessimistic locking?
    - **Answer:** Pessimistic locking assumes conflicts are likely and locks resources upfront using `WITH (UPDLOCK)` or `SELECT ... FOR UPDATE`. I used UPDLOCK hints in the critical section of the inventory allocation to prevent double-allocation of the same serial number.
    - **If asked more:** I would explain the trade-off: pessimistic locking reduces throughput but guarantees correctness, which was acceptable for low-concurrency critical operations.
51. What is connection pooling?
    - **Answer:** Connection pooling reuses database connections instead of creating new ones per request. In our Spring Boot application, I configured HikariCP with a max pool size of 20, tuned based on the number of concurrent API requests and database capacity.
    - **If asked more:** I would explain how I diagnosed connection exhaustion by monitoring active connections in MSSQL and increased the pool size gradually while watching for contention.
52. How do you optimize a slow query?
    - **Answer:** I start by examining the actual execution plan for table scans, high-cost operators, and missing index hints. Then I check indexes, rewrite non-sargable conditions, and add INCLUDE columns. In CDMS, I systematically applied this to each subquery of the 5-hour procedure.
    - **If asked more:** I would explain my full playbook: SET STATISTICS TIME/IO ON, check wait stats, look for parameter sniffing issues, and test with `OPTION (RECOMPILE)` if needed.
53. How do you optimize a stored procedure?
    - **Answer:** I profile the procedure by running it with `SET STATISTICS TIME ON` and capturing the actual plan, identify the most expensive statements, optimize them individually, and then re-run the whole procedure to confirm the cumulative improvement. That's exactly how I brought CDMS from 5 hours to 12 minutes.
    - **If asked more:** I would explain how I parameterized the procedure to prevent plan caching issues, and how I used `WITH RECOMPILE` for procedures with highly varying parameters.
54. How do you handle large data processing?
    - **Answer:** I use batch processing with set-based operations, avoid cursors and row-by-row processing, and ensure proper indexing. In the inventory system, I processed 10,000+ serial records in batches of 1,000 using `INSERT ... SELECT` with `ROWNUMBER()` filtering.
    - **If asked more:** I would explain the difference between batch size tuning — too small causes many round-trips, too large causes lock escalation — based on what I observed during load testing.
55. What is pagination in SQL?
    - **Answer:** Pagination retrieves a subset of rows using `OFFSET` and `FETCH NEXT` in MSSQL. In the CDMS partner list API, I paginated results with `ORDER BY PartnerID OFFSET 0 ROWS FETCH NEXT 50 ROWS ONLY` to avoid loading thousands of partners at once.
    - **If asked more:** I would explain how offset pagination degrades with large offsets because the engine still reads all skipped rows, and when I'd switch to keyset pagination.
56. Difference between offset pagination and keyset pagination.
    - **Answer:** Offset pagination skips N rows using `OFFSET`; keyset pagination uses `WHERE id > @lastSeenId`. Offset pagination becomes slow on large datasets because it reads all skipped rows each time. In the inventory audit log, I switched to keyset pagination for faster queries.
    - **If asked more:** I would show how keyset pagination is ideal for infinite scroll on sorted unique columns but can't skip to arbitrary pages.
57. What is query parameterization?
    - **Answer:** Parameterization uses placeholders instead of hardcoded values in SQL, which allows plan reuse and prevents SQL injection. In our Spring Boot JPA repositories, all queries were automatically parameterized via prepared statements.
    - **If asked more:** I would explain how ad-hoc queries without parameterization caused plan cache bloat in CDMS, and how I forced parameterization using `ALTER DATABASE SET PARAMETERIZATION FORCED`.
58. What is SQL injection?
    - **Answer:** SQL injection is an attack where malicious SQL is inserted through user input. In the inventory API, I prevented this by never concatenating user input into SQL strings — all queries used JPA repositories or parameterized stored procedures.
    - **If asked more:** I would demonstrate how string concatenation like `"WHERE name = '" + userInput + "'"` is vulnerable and how parameterized queries like `WHERE name = @name` prevent it.
59. How do you prevent SQL injection?
    - **Answer:** Always use parameterized queries, stored procedures, or ORM frameworks that handle escaping. In all my projects, I strictly use Spring Data JPA or `@Query` with named parameters, and I validate inputs at the controller layer before they reach the database.
    - **If asked more:** I would explain that stored procedures with `EXEC` inside can still be vulnerable, so I never build dynamic SQL inside procedures and instead use CASE statements or multiple queries.
60. What is ETL?
    - **Answer:** ETL (Extract, Transform, Load) is the process of moving data from source systems to a target database. In CDMS, I worked on an SSIS-based ETL pipeline that extracted partner data from various formats, transformed it (cleaning, validating, aggregating), and loaded it into MSSQL reporting tables.
    - **If asked more:** I would walk through a specific ETL flow: extracting CSV files from partner uploads, transforming serial numbers to a standard format, and loading them into the inventory tables with error logging.
61. What is SSIS?
    - **Answer:** SSIS (SQL Server Integration Services) is Microsoft's ETL tool for data migration and integration. In CDMS, I used SSIS packages to automate the nightly data load from partner files into the staging and reporting tables.
    - **If asked more:** I would explain how I used SSIS data flow tasks with lookup transformations to validate partner codes against the reference table, redirecting invalid rows to an error output.
62. What is data validation in ETL?
    - **Answer:** Data validation in ETL ensures that transformed data meets business rules before loading. In the inventory ETL, I validated serial number formats, partner codes against the master list, and date ranges before inserting into the target table.
    - **If asked more:** I would explain the validation layers: schema validation, business rule validation, and referential integrity checks — and how I logged each failure type separately for partner reporting.
63. How do you handle duplicate records?
    - **Answer:** I use `ROW_NUMBER()` with `PARTITION BY` to identify duplicates, then decide whether to deduplicate (keep first/latest) or reject. In the inventory pipeline, I partitioned by serial number and kept the record with the latest timestamp, logging the rejected duplicates for partner notification.
    - **If asked more:** I would explain the business trade-off: sometimes duplicates should be rejected (unique serials), sometimes merged (duplicate partner submissions), depending on the data domain.
64. How do you handle missing data?
    - **Answer:** For missing data, I first check if it's truly required or can be defaulted. In the inventory ETL, if a serial record was missing the partner ID, I logged it to an error table and skipped it because we couldn't determine ownership without the partner.
    - **If asked more:** I would explain the difference between NULL handling strategies: COALESCE for defaults, ISNULL checks, and how missing required fields should fail fast rather than propagate bad data.
65. How do you reconcile source and target data?
    - **Answer:** Reconciliation compares row counts, checksums, or key values between source and target after ETL. In CDMS, I built a reconciliation script using `EXCEPT` and `INTERSECT` operators to find mismatches between the source partner files and the loaded reporting tables.
    - **If asked more:** I would explain how I automated reconciliation as the final step of the ETL pipeline, sending an email with the mismatch count so the team could investigate before downstream reports were generated.

---

## Kafka and Messaging Questions

1. What is Kafka?
   - **Answer:** Kafka is a distributed event-streaming platform used for high-throughput, fault-tolerant data pipelines. In the cold-chain project, Kafka was the backbone that ingested real-time temperature sensor data from IoT gateways and made it available to multiple downstream consumers.
   - **If asked more:** I would explain the publish-subscribe model and contrast Kafka with traditional message queues — Kafka's log-based storage allows replay, which was critical for backfilling historical temperature data.
2. Why use Kafka?
   - **Answer:** Kafka handles high throughput, provides durability through disk-based logs, and allows multiple consumers to process the same stream independently. We chose Kafka in the cold-chain project because IoT gateways could burst thousands of sensor readings per second and we needed a buffer that could absorb the spikes without data loss.
   - **If asked more:** I would compare Kafka with alternatives (RabbitMQ, SQS) and explain why Kafka's replay capability was essential for debugging temperature excursion incidents.
3. What is event streaming?
   - **Answer:** Event streaming captures data in real-time as a sequence of events. In the cold-chain system, every temperature reading from each sensor was an event published to Kafka, creating an immutable log that we could process, analyze, and visualize in real-time on the Grafana dashboard.
   - **If asked more:** I would explain how event streaming differs from request-response — events are persisted, can be replayed, and allow multiple independent consumers to process the same data.
4. What is producer?
   - **Answer:** A producer publishes messages to Kafka topics. In our cold-chain pipeline, an AWS Lambda function acted as the producer — it received MQTT messages from AWS IoT Core and published them to the `sensor-readings` Kafka topic on our EC2-hosted Kafka cluster.
   - **If asked more:** I would discuss producer configuration — how I set `acks=all` to ensure no data loss and how I tuned `linger.ms` and `batch.size` for throughput.
5. What is consumer?
   - **Answer:** A consumer subscribes to topics and processes messages. In the cold-chain project, our Spring Boot application was the consumer — it read temperature readings from Kafka, validated them, and stored them in InfluxDB for time-series analysis.
   - **If asked more:** I would explain consumer offset management, auto-commit vs manual commit, and how I configured the consumer for at-least-once delivery with idempotent processing.
6. What is topic?
   - **Answer:** A topic is a logical channel where producers send messages and consumers read from. In the cold-chain system, I organized topics by data type: `sensor-readings` for temperature data, `sensor-alerts` for threshold violations, and `sensor-health` for gateway heartbeat messages.
   - **If asked more:** I would explain how I used topic naming conventions and considered the number of topics based on data volume and consumer isolation requirements.
7. What is partition?
   - **Answer:** A partition is a unit of parallelism within a topic — messages are distributed across partitions, which are ordered and immutable. I configured the `sensor-readings` topic with 6 partitions to match the 6 IoT gateways, using the gateway ID as the message key to ensure all readings from one gateway went to the same partition.
   - **If asked more:** I would explain how partition count affects consumer parallelism — more partitions allow more consumers, but too many increase overhead and rebalancing time.
8. What is broker?
   - **Answer:** A broker is a Kafka server that stores data and serves clients. We ran a 3-broker Kafka cluster on EC2 instances, which gave us fault tolerance — if one broker failed, the others continued serving with replicated data.
   - **If asked more:** I would explain how we sized the brokers based on expected throughput and retention — we calculated disk space based on 7 days of sensor data at 1000 msg/sec with a replication factor of 3.
9. What is consumer group?
   - **Answer:** A consumer group is a set of consumers that collectively read from a topic, with each partition assigned to one consumer. In cold-chain, we had multiple consumer instances in the same group to parallelize processing of sensor data from different gateways.
   - **If asked more:** I would explain consumer group rebalancing — what happens when a consumer joins or leaves, and how we tuned `session.timeout.ms` to reduce unnecessary rebalances during GC pauses.
10. How does Kafka scale?
    - **Answer:** Kafka scales horizontally by adding brokers to the cluster and partitions to topics. More partitions allow more consumers in a group to process in parallel. In cold-chain, when we added two more gateways, I increased the partition count and added consumer instances to handle the increased load.
    - **If asked more:** I would explain the difference between scaling producers (they write to any partition) and consumers (limited by partition count), and how I used Kafka's built-in rebalancing to distribute load automatically.
11. Why are partitions important?
    - **Answer:** Partitions enable parallelism — multiple consumers can read different partitions concurrently. Without partitions, we'd have a single sequential stream. In cold-chain, 6 partitions meant 6 consumers could process temperature data in parallel, keeping the pipeline responsive even during sensor bursts.
    - **If asked more:** I would explain how partitions provide ordering guarantees (within a partition) and how the number of partitions is the maximum parallelism for consumers.
12. How do you decide partition count?
    - **Answer:** I base partition count on the expected throughput, number of consumers, and key cardinality. For cold-chain, I started with 6 partitions — one per gateway — and monitored consumer lag. I planned to increase to 12 if lag grew, but 6 was sufficient for our 1000 msg/sec peak.
    - **If asked more:** I would share the formula: partitions should be at least max(target throughput / partition throughput, number of consumers). I also consider that too many partitions increase ZooKeeper overhead and rebalancing time.
13. What is offset?
    - **Answer:** An offset is a sequential ID assigned to each message within a partition, representing its position. Offsets allow consumers to track how far they've read. In the cold-chain pipeline, if a consumer crashed, it could resume from its last committed offset without missing or duplicating messages.
    - **If asked more:** I would explain how offset commits work — auto-commit at `enable.auto.commit=true` with `auto.commit.interval.ms`, and why I switched to manual commits for exactly-once semantics in critical alert processing.
14. What is committed offset?
    - **Answer:** A committed offset is the last offset a consumer has successfully processed and saved to Kafka's internal `__consumer_offsets` topic. When the inventory consumer crashed during a peak load, the committed offset ensured it resumed from where it left off rather than re-processing thousands of messages.
    - **If asked more:** I would explain the risk of committing before processing (message loss on crash) vs after processing (duplicate reprocessing), and how I accepted duplicates in favor of zero data loss.
15. What is consumer lag?
    - **Answer:** Consumer lag is the difference between the latest offset in a partition and the consumer's committed offset. When our InfluxDB write path had a slowdown, I saw lag spike to 50,000 messages. That told me the consumer was falling behind and couldn't keep up with the producer rate.
    - **If asked more:** I would explain how I monitored lag using `kafka-consumer-groups` CLI and set up CloudWatch alarms on lag metrics to get paged before the pipeline fell too far behind.
16. How do you monitor consumer lag?
    - **Answer:** I used the `kafka-consumer-groups --bootstrap-server --describe --group` command to check lag per partition. In cold-chain, I automated this by publishing lag metrics to CloudWatch every minute and set a threshold alarm at 10,000 messages lag for immediate investigation.
    - **If asked more:** I would also mention Burrow, a LinkedIn tool for lag monitoring, and how lag trends (increasing vs stable) tell you whether the consumer is catching up or falling further behind.
17. What is replication factor?
    - **Answer:** Replication factor determines how many copies of each partition exist across brokers. In cold-chain, I set replication factor to 3 for the sensor-readings topic so that if one EC2 broker instance went down, no data was lost and the cluster continued serving.
    - **If asked more:** I would explain the trade-off: higher replication provides better durability but uses more disk and network bandwidth. RF=3 with min.insync.replicas=2 was our standard for production.
18. What is leader and follower replica?
    - **Answer:** For each partition, one broker is the leader that handles all reads and writes, and the rest are in-sync followers. When the leader broker for our sensor-readings partition went down for maintenance, a follower automatically became the new leader with zero data loss.
    - **If asked more:** I would explain how Kafka handles leader election and how the preferred replica configuration helps redistribute leadership after a failed broker recovers.
19. What is ISR?
    - **Answer:** ISR (In-Sync Replicas) are followers that are fully caught up with the leader. I set `min.insync.replicas=2` for cold-chain topics to ensure that at least 2 brokers acknowledged each write. This prevented data loss even if one broker failed after acknowledging.
    - **If asked more:** I would explain what happens when a follower falls out of ISR (slow network, GC pause) and how it catches up using truncated replication.
20. What happens when a broker fails?
    - **Answer:** When a broker fails, the controllers detect it, partition leaders on that broker are reassigned to ISR followers on other brokers. During one EC2 instance failure in cold-chain, the Kafka cluster automatically reassigned the leadership within seconds, and producers/consumers continued with minimal interruption.
    - **If asked more:** I would explain the difference between a clean shutdown (controlled leader migration) vs a hard failure (unclean leader election if no ISR is available), and how we handled each case.
21. What is acknowledgement in Kafka?
    - **Answer:** Acknowledgement (acks) determines when the producer considers a write successful. In cold-chain, I used `acks=all` for the temperature topic because losing a reading was unacceptable — the producer waited for all in-sync replicas to acknowledge before moving on.
    - **If asked more:** I would explain the performance impact of `acks=all` (higher latency, guaranteed durability) and how we tuned `max.in.flight.requests.per.connection` to prevent ordering issues.
22. Difference between `acks=0`, `acks=1`, and `acks=all`.
    - **Answer:** `acks=0` fires and forgets (fastest, highest risk), `acks=1` waits for the leader only (balanced), `acks=all` waits for all ISRs (safest, slower). I used `acks=all` for sensor data and `acks=1` for non-critical health check messages.
    - **If asked more:** I would explain how these interact with `min.insync.replicas` and why `acks=all` with `min.insync.replicas=2` still works even if one broker is down.
23. What is at-most-once delivery?
    - **Answer:** At-most-once means a message is delivered zero or one time — if the consumer fails after fetching but before processing, the message is lost. In cold-chain, I avoided this for temperature data because losing a reading could mean missing a temperature excursion.
    - **If asked more:** I would explain the configuration: `enable.auto.commit=true` with commits before processing, combined with `acks=0` on the producer side.
24. What is at-least-once delivery?
    - **Answer:** At-least-once means messages can be delivered more than once but never lost. In cold-chain, I used at-least-once by setting `enable.auto.commit=false` and manually committing offsets only after successful processing and storage in InfluxDB.
    - **If asked more:** I would explain that this requires idempotent consumers (since duplicates can happen), and how I implemented deduplication using event IDs in the consumer.
25. What is exactly-once semantics?
    - **Answer:** Exactly-once semantics (EOS) ensures each message is processed exactly once, with no duplicates and no losses. While Kafka supports EOS with transactions and idempotent producers, I used at-least-once with idempotent consumers instead because the setup complexity was higher than the duplicate probability.
    - **If asked more:** I would explain the trade-offs: EOS adds latency and requires transactional coordination, so it's worth it for financial systems but overkill for IoT sensor data where occasional duplicates are tolerable.
26. How do you handle duplicate messages?
    - **Answer:** I handle duplicates by making consumers idempotent — either by tracking processed event IDs in Redis or using database unique constraints. In cold-chain, each sensor reading had a unique ID (gatewayID + timestamp), and I used a unique constraint on that in InfluxDB to prevent duplicate storage.
    - **If asked more:** I would explain the difference between producer-side duplicates (from retries) and consumer-side duplicates (from rebalancing), and how I handled both.
27. How do you make consumers idempotent?
    - **Answer:** Idempotent consumers produce the same result regardless of how many times a message is processed. In the cold-chain consumer, I checked if the unique event ID already existed in the target table before inserting, and skipped duplicates. This made the entire pipeline safe to retry during failures.
    - **If asked more:** I would discuss the trade-off between exactly-once (Kafka transactions) and idempotent consumers, and why I chose idempotency for simplicity.
28. What is dead-letter topic?
    - **Answer:** A dead-letter topic (DLT) stores messages that consumers cannot process after repeated retries. In cold-chain, I configured a DLT for the alert-processing topic — if a sensor reading failed validation after 3 retries, it was moved to the DLT for manual inspection rather than blocking the consumer stream.
    - **If asked more:** I would explain how I set up the DLT with a TTL and automated alerting to notify the team when messages landed there, so we could investigate the root cause.
29. What is retry topic?
    - **Answer:** A retry topic holds messages that failed temporarily so they can be reprocessed after a delay. In cold-chain, when InfluxDB was temporarily unavailable, the consumer moved the message to a retry topic with a 10-second delay before the next attempt, preventing a tight retry loop.
    - **If asked more:** I would explain the retry topic architecture: main topic → consumer → retry topic → retry consumer → DLT (if still failing), and how to set the retry delay.
30. How do you handle poison messages?
    - **Answer:** Poison messages are messages that always cause consumer failures. In cold-chain, a malformed JSON payload from a faulty gateway once caused continuous deserialization errors. I handled it by wrapping the deserialization in a try-catch and sending the bad message to a DLT with the error details logged.
    - **If asked more:** I would explain how to detect poison messages — repeated failures on the same offset — and why automated DLT routing is critical to prevent the consumer from being stuck in a rebalance loop.
31. What is message ordering?
    - **Answer:** Message ordering guarantees that messages are processed in the order they were produced. Kafka guarantees order within a partition, not across partitions. In cold-chain, I used the gateway ID as the message key so all readings from one gateway went to the same partition, preserving per-gateway ordering.
    - **If asked more:** I would explain the trade-off: strong ordering limits parallelism (key → single partition), so I designed around it by keeping per-gateway ordering but allowing different gateways to be processed out of order.
32. How do you guarantee ordering in Kafka?
    - **Answer:** Use the same message key for all related messages — Kafka assigns messages with the same key to the same partition. In cold-chain, `gatewayId` was the key, so temperature readings from each gateway were strictly ordered. I also set `max.in.flight.requests.per.connection=1` to prevent reordering on producer retries.
    - **If asked more:** I would explain the impact of `enable.idempotence=true` on ordering — Kafka internally handles ordering with idempotent producers without `max.in.flight.requests=1`.
33. What is key in Kafka message?
    - **Answer:** The message key determines which partition a message goes to — same key always goes to the same partition. In cold-chain, the key was the gateway device ID, which ensured all sensor data from one gateway was in order and also helped with debugging by filtering messages by gateway.
    - **If asked more:** I would explain how null keys cause round-robin distribution across partitions (no ordering guarantee), and how I chose keys based on ordering vs. load-balancing needs.
34. What is schema registry?
    - **Answer:** Schema Registry stores and validates message schemas (Avro/JSON Schema) to ensure compatibility between producers and consumers. While we used JSON with a shared contract in cold-chain, Schema Registry would have prevented issues like a producer sending an extra field without the consumer expecting it.
    - **If asked more:** I would explain backward/forward compatibility and how Schema Registry helps with evolution — a producer can add optional fields without breaking existing consumers.
35. What is Kafka retention?
    - **Answer:** Retention controls how long Kafka retains messages. In cold-chain, I set retention to 7 days for the sensor-readings topic. This allowed us to reprocess data if a downstream system failed over the weekend without needing to contact the gateways again.
    - **If asked more:** I would explain the difference between time-based retention and size-based retention, and how log compaction retains only the latest message per key for stateful streams.
36. Difference between Kafka and RabbitMQ.
    - **Answer:** Kafka is a distributed log optimized for high-throughput event streaming with replay capability; RabbitMQ is a traditional message broker with complex routing and priority queues. I chose Kafka for cold-chain because the IoT data was high-volume, needed replay, and had multiple consumers, which RabbitMQ handles less efficiently.
    - **If asked more:** I would argue that RabbitMQ is better for task distribution (commands, work queues) while Kafka is better for event streaming, and the choice depends on whether you need message deletion after consumption or log retention.
37. Difference between Kafka and SQS.
    - **Answer:** SQS is a fully managed AWS queue with at-least-once delivery and automatic scaling; Kafka requires manual cluster management but offers lower latency, higher throughput, and multi-consumer fan-out. We used Kafka on EC2 because we needed sub-100ms latency for real-time dashboards and multiple consumer groups — SQS would have added complexity for fan-out.
    - **If asked more:** I would explain when SQS wins: simpler setup, no infrastructure management, automatic scaling, and when throughput requirements are moderate (under 10K msg/sec).
38. Difference between Kafka and MQTT.
    - **Answer:** MQTT is a lightweight pub-sub protocol for IoT devices with limited bandwidth; Kafka is a distributed storage and streaming platform. In cold-chain, they complemented each other: MQTT carried data from gateways to AWS IoT Core, and Kafka handled the backend streaming from IoT Core to the data stores.
    - **If asked more:** I would explain how MQTT's QoS levels (0, 1, 2) map to Kafka's delivery semantics, and why we chose QoS 1 (at-least-once) for MQTT combined with Kafka's at-least-once for end-to-end reliability.
39. When would you not use Kafka?
    - **Answer:** I would not use Kafka for simple task queues, low-throughput (< 100 msg/sec) scenarios, or when you need exactly-once delivery without idempotent consumers. Also, if the team lacks operational experience with Kafka, the learning curve and operational overhead can be significant.
    - **If asked more:** I would give a concrete example: if we only had to process 10 temperature readings per minute and didn't need replay, a simple REST API call would be simpler than running a Kafka cluster on EC2.
40. How did you use Kafka in your cold-chain project?
    - **Answer:** Kafka was the central message backbone. IoT gateways sent temperature data via MQTT to AWS IoT Core, which forwarded it to Lambda. Lambda published the data to our Kafka topic (`sensor-readings`) running on EC2. A Spring Boot consumer read from Kafka, validated readings, and stored them in InfluxDB. A separate consumer handled threshold-based alerting.
    - **If asked more:** I would draw the complete architecture: Gateway → IoT Core → Lambda (producer) → Kafka (brokers on EC2) → Spring Boot (consumer) → InfluxDB + another consumer → Grafana alerts.
41. Why was Kafka useful for IoT data?
    - **Answer:** IoT data is high-volume, continuous, and comes from many sources. Kafka's log-based architecture absorbed bursty sensor data without backpressure, provided a durable buffer that prevented data loss during downstream outages, and allowed multiple consumers (storage, alerting, analytics) to process the same stream independently.
    - **If asked more:** I would mention that if InfluxDB went down for 30 minutes, Kafka's 7-day retention meant zero data loss — the consumer just caught up when InfluxDB was back, which wouldn't be possible with a point-to-point integration.
42. How do you handle sensor data bursts in Kafka?
    - **Answer:** Kafka naturally handles bursts because it's disk-based — producers can write faster than consumers can read. However, to prevent unbounded lag, I sized the partition count and consumer instances to handle peak throughput. I also set producer `max.block.ms` and consumer `fetch.max.bytes` appropriately to avoid memory issues during bursts.
    - **If asked more:** I would explain the concept of backpressure in Kafka — it doesn't apply to producers (they always write), but consumers can apply backpressure by pausing partitions via `consumer.pause()` when downstream systems are slow.
43. How do you scale Kafka consumers?
    - **Answer:** You scale consumers by adding more consumer instances to the same consumer group. The maximum parallelism equals the number of partitions. In cold-chain, when we increased from 3 to 6 gateways, I increased the partition count from 3 to 6 and added 3 more consumer instances, distributing the load.
    - **If asked more:** I would warn that increasing partitions after topic creation creates a new partition assignment, and existing messages aren't redistributed — so plan the partition count upfront based on projected growth.
44. How do you secure Kafka?
    - **Answer:** Kafka can be secured with SSL/TLS for encryption, SASL for authentication, and ACLs for authorization. In the cold-chain project, since Kafka ran on EC2 within a VPC and was accessed only by internal services, we used security groups for network isolation rather than enabling Kafka-level SSL.
    - **If asked more:** I would explain how to set up SASL/SCRAM for username-password auth, and why you should always encrypt data in transit if Kafka is accessible outside the VPC.
45. What is Kafka Connect?
    - **Answer:** Kafka Connect is a framework for streaming data between Kafka and external systems using connectors. I evaluated the InfluxDB Sink Connector for cold-chain to replace the custom Spring Boot consumer, but the connector didn't support all our validation logic, so I kept the custom consumer.
    - **If asked more:** I would explain the difference between source connectors (push data into Kafka) and sink connectors (pull data from Kafka), and how they simplify integration without writing producer/consumer code.
46. What is Kafka Streams?
    - **Answer:** Kafka Streams is a Java library for building stream processing applications on top of Kafka. In cold-chain, I considered using Kafka Streams for the sliding-window temperature average calculation, but implemented it in the Spring Boot consumer instead due to familiarity.
    - **If asked more:** I would explain stateful operations (windowed aggregations, joins) and how Kafka Streams handles state using RocksDB for local state stores with changelog topics for fault tolerance.
47. What is backpressure?
    - **Answer:** Backpressure is a feedback mechanism where a slow consumer signals the producer to slow down. Kafka doesn't have traditional backpressure — producers always write at full speed, and consumers fall behind if they can't keep up. In cold-chain, the downstream InfluxDB write path created backpressure on the consumer, causing lag.
    - **If asked more:** I would explain how I handled this by adding a processing buffer with bounded queues in the consumer and using `consumer.pause()` when the buffer was full, resuming when it drained.
48. How do you handle backpressure?
    - **Answer:** In Kafka, you handle backpressure at the consumer level by pausing partition assignment when the processing pipeline is saturated. In cold-chain, when InfluxDB had a write slowdown, the consumer paused all partitions, waited for pending writes to complete, and then resumed consuming.
    - **If asked more:** I would explain the alternative approach: using a bounded queue between the poll loop and the processing thread pool, and comparing Kafka's approach to reactive streams backpressure.
49. How do you test Kafka consumers?
    - **Answer:** I test Kafka consumers using embedded Kafka with Spring Kafka's `@EmbeddedKafka` annotation in integration tests. In cold-chain, I wrote tests that published sample sensor data to an embedded topic, verified the consumer processed it, and checked the data landed in a test InfluxDB instance.
    - **If asked more:** I would also mention testing failure scenarios: broker failure, consumer rebalancing, poison messages, and verifying that offsets are committed correctly after processing.
50. How do you debug message loss?
    - **Answer:** I start by checking consumer lag — if lag is zero, messages were consumed. Then I check the consumer's committed offset against the latest offset. I also enable producer-side logging with `errors.tolerance=all` and check the DLT. In cold-chain, one message loss was caused by a consumer crash between processing and offset commit.
    - **If asked more:** I would describe a systematic debugging approach: check producer acks, consumer error logs, offset commits, and then replay the topic to verify. The key insight is knowing the delivery semantic and checking each layer.

---

## AWS and Cloud Questions

1. What AWS services have you used?
   - **Answer:** I've used EC2 (Spring Boot + Kafka deployment), IoT Core (MQTT gateway ingestion), Lambda (lightweight data transformation), S3 (report storage), IAM (access control), and CloudWatch (monitoring and logging). Each was chosen for a specific role in the cold-chain and CDMS projects.
   - **If asked more:** I would go service by service — why I chose EC2 over Lambda for Kafka, how IoT Core handled device authentication, and how CloudWatch alarms alerted us to temperature excursions.
2. What is EC2?
   - **Answer:** EC2 is AWS's virtual server service. We deployed our Spring Boot APIs and Kafka brokers on EC2 instances, configured security groups for network access, and used auto-scaling to handle load. For Kafka, we chose EC2 over MSK to have full control over broker configuration.
   - **If asked more:** I would explain the instance sizing decisions — we used m5.large for Spring Boot and m5.xlarge for Kafka brokers based on memory and I/O requirements.
3. What is S3?
   - **Answer:** S3 is AWS's object storage service for storing and retrieving any amount of data. In CDMS, I used S3 to store generated reports — the optimized stored procedure output was written to S3 as CSV files that partners could download through the portal.
   - **If asked more:** I would discuss S3 storage classes — we used Standard for recent reports and transitioned older ones to Glacier for cost savings.
4. What is Lambda?
   - **Answer:** Lambda is AWS's serverless compute service that runs code on demand. In the cold-chain pipeline, I used a Lambda function to transform MQTT messages from AWS IoT Core and publish them to Kafka. The Lambda was triggered by an IoT Core rule and ran for under a second per invocation.
   - **If asked more:** I would explain how I configured the Lambda's memory (256MB) and timeout (30 seconds) based on the average message size, and how I handled cold start by keeping a warm pool.
5. What is AWS IoT Core?
   - **Answer:** AWS IoT Core is a managed cloud service that lets IoT devices connect and interact with AWS applications via MQTT. In cold-chain, IoT Core authenticated each gateway using device certificates, received temperature readings via MQTT, and routed them to Lambda through a rule.
   - **If asked more:** I would explain how IoT Core's device shadows stored the last known state of each gateway, which helped detect when a gateway went offline unexpectedly.
6. Why use AWS IoT Core?
   - **Answer:** We used IoT Core because it handles the heavy lifting of MQTT broker management, device authentication via X.509 certificates, and scales automatically with the number of connected gateways. Setting up a custom MQTT broker on EC2 would have required more operational effort.
   - **If asked more:** I would compare IoT Core with running a self-managed Mosquitto broker — IoT Core's device registry and policy engine saved us from building device management from scratch.
7. How does MQTT work with AWS IoT Core?
   - **Answer:** IoT gateways establish persistent MQTT connections to AWS IoT Core using device certificates. The gateways publish temperature readings to a topic like `sensors/gateway1/temperature`, and IoT Core triggers a rule that forwards the message to Lambda for further processing.
   - **If asked more:** I would explain MQTT QoS levels — we used QoS 1 (at-least-once) for reliable delivery — and how IoT Core's topic filters allowed us to subscribe to all gateways with a wildcard pattern.
8. Difference between EC2 and Lambda.
   - **Answer:** EC2 provides full control over the OS and runtime, suitable for stateful or long-running services like Kafka; Lambda is ephemeral and event-driven, ideal for short stateless tasks. I chose EC2 for Kafka because it needs persistent storage and continuous operation, and Lambda for message transformation because it runs only when data arrives.
   - **If asked more:** I would explain how Lambda's 15-minute timeout and 10GB storage limit made it unsuitable for Kafka, which needs sustained throughput and disk I/O.
9. When would you use Lambda?
   - **Answer:** Use Lambda for event-driven, short-lived tasks like transforming data between services, resizing images, or responding to API Gateway requests. In cold-chain, Lambda was perfect for converting MQTT messages to Kafka-compatible JSON — a simple, stateless transformation that ran for milliseconds per event.
   - **If asked more:** I would explain when Lambda is not a good fit: long-running processes (>15 min), stateful processing, or high-throughput sustained workloads where cost becomes unpredictable.
10. When would you avoid Lambda?
    - **Answer:** Avoid Lambda for stateful services, long-running computations, or workloads requiring consistent low latency. In cold-chain, we avoided Lambda for the main data processing pipeline because we needed sustained processing of 1000+ msg/sec and Kafka consumers are better suited for that.
    - **If asked more:** I would also mention cost — Lambda can be more expensive than EC2 for high-utilization workloads, and cold starts add latency that is unacceptable for real-time dashboards.
11. What is serverless?
    - **Answer:** Serverless means you don't manage servers — AWS handles scaling, patching, and availability. Lambda, IoT Core, and S3 are serverless services we used. Serverless is great for variable workloads because you pay only for what you use, but less suitable for predictable, high-utilization services like Kafka.
    - **If asked more:** I would explain the trade-off: serverless reduces operational overhead but can lead to unpredictable costs and cold-start latency, so we used a hybrid approach (serverless for ingestion, EC2 for processing).
12. What is IAM?
    - **Answer:** IAM (Identity and Access Management) controls who can access AWS resources and what they can do. In cold-chain, I created IAM roles with least-privilege policies — Lambda had permission only to publish to the Kafka topic, and EC2 instances had access only to CloudWatch logs.
    - **If asked more:** I would explain the difference between IAM users (for people) and IAM roles (for services), and how I debugged access denied errors using CloudTrail.
13. What is IAM role?
    - **Answer:** An IAM role is an identity that AWS services assume to get temporary permissions. In cold-chain, the Lambda function had an IAM role with policies that allowed it to receive messages from IoT Core and publish to the Kafka topic. This avoided hardcoding any credentials.
    - **If asked more:** I would explain how I configured the trust policy to allow Lambda to assume the role, and how temporary credentials are rotated automatically by AWS.
14. What is IAM policy?
    - **Answer:** An IAM policy defines specific permissions in JSON format. I wrote a policy that allowed the Lambda to `kafka-cluster:Connect` and `kafka-cluster:WriteData` only to the `sensor-readings` topic, following the principle of least privilege.
    - **If asked more:** I would walk through a sample policy and explain how I tested it using IAM Policy Simulator before applying it to production roles.
15. What is VPC?
    - **Answer:** VPC (Virtual Private Cloud) is a logically isolated network within AWS. Our Kafka brokers and Spring Boot services ran inside a VPC with private subnets, so they were not accessible from the public internet. Only the Lambda function had access through a VPC endpoint.
    - **If asked more:** I would explain the VPC components — subnets, route tables, NAT gateways — and how we set up a public subnet for the load balancer and private subnets for the application tier.
16. What is subnet?
    - **Answer:** A subnet is a range of IP addresses within a VPC. In cold-chain, we used public subnets for the load balancer and private subnets for EC2 instances running Kafka and Spring Boot. The private subnets had no direct internet access, adding a security layer.
    - **If asked more:** I would explain the difference between public and private subnets — public subnets have a route to the internet gateway, private subnets route through NAT for outbound access only.
17. What is security group?
    - **Answer:** A security group acts as a virtual firewall for EC2 instances. I configured security groups to allow inbound traffic on port 9092 only from the Spring Boot application's security group, and port 8080 only from the load balancer. This micro-segmentation prevented direct access to services.
    - **If asked more:** I would explain stateful vs stateless firewalls — security groups are stateful, so return traffic is automatically allowed, unlike NACLs which are stateless.
18. What is load balancer?
    - **Answer:** A load balancer distributes incoming traffic across multiple EC2 instances for high availability and fault tolerance. In CDMS, we used an Application Load Balancer in front of the Spring Boot API instances to handle request routing, health checks, and SSL termination.
    - **If asked more:** I would explain how we configured the target groups, health check endpoints, and stickiness — we didn't need session stickiness because JWT tokens carried all session state.
19. What is auto scaling?
    - **Answer:** Auto Scaling automatically adjusts the number of EC2 instances based on demand. For the cold-chain Spring Boot API, I set a scale-out policy at 70% CPU and a scale-in policy at 30%, with a minimum of 2 and maximum of 6 instances to handle traffic spikes during temperature excursion events.
    - **If asked more:** I would explain the cooldown period and how we tested auto scaling with a load generator that simulated multiple dashboard users querying the API simultaneously.
20. What is CloudWatch?
    - **Answer:** CloudWatch is AWS's monitoring service for logs, metrics, and alarms. In the cold-chain project, I published custom metrics (consumer lag, processing rate, gateway status) to CloudWatch, set up dashboards, and configured alarms to email the team when temperature excursions were detected.
    - **If asked more:** I would explain how I used CloudWatch Logs to centralize logs from all EC2 instances and Lambda functions, and how I set up log group retention policies to save costs.
21. How do you monitor AWS applications?
    - **Answer:** I use CloudWatch for infrastructure metrics (CPU, memory, disk), custom application metrics published via the CloudWatch agent, and centralized logging with CloudWatch Logs. In cold-chain, I also configured detailed monitoring on EC2 and set up composite alarms that considered multiple metrics before paging the team.
    - **If asked more:** I would explain the difference between basic monitoring (5-minute intervals) and detailed monitoring (1-minute intervals), which I enabled for critical Kafka brokers.
22. How do you store files in S3?
    - **Answer:** I use the AWS SDK's `PutObjectRequest` to upload files from Spring Boot to S3. In CDMS, after the stored procedure generated the report CSV, the application uploaded it to an S3 bucket and returned a pre-signed URL that partners could use to download the report securely.
    - **If asked more:** I would explain pre-signed URLs — how I generated them with a 24-hour expiration, and how the bucket policy restricted access to only the application's IAM role.
23. What is S3 bucket policy?
    - **Answer:** An S3 bucket policy defines who can access the bucket and what operations they can perform. In CDMS, I wrote a bucket policy that allowed only the application's IAM role to upload files and allowed only authenticated partners (via pre-signed URL) to download specific objects.
    - **If asked more:** I would explain how I used bucket policies to enforce encryption in transit (AWS:SecureTransport) and deny public access at the bucket level.
24. How do you secure S3?
    - **Answer:** I secure S3 by blocking public access at the bucket level, using IAM policies for access control, enabling server-side encryption (SSE-S3), and using pre-signed URLs for temporary access. I also enabled S3 access logs to audit all requests and set up CloudWatch alarms for unusual access patterns.
    - **If asked more:** I would explain the shared responsibility model — AWS secures the infrastructure, I secure my buckets through proper configuration and access policies.
25. How do you deploy Spring Boot on EC2?
    - **Answer:** I build the Spring Boot application into a JAR using Maven, Dockerize it, push the image to Docker Hub, and Jenkins pulls the image on the EC2 instance and runs the container. The EC2 instance has the Docker runtime and environment variables configured via the user data script.
    - **If asked more:** I would explain the deployment script — how Jenkins connected to EC2 via SSH, stopped the old container, pulled the new image, and started the new container with health check verification.
26. How do you run Kafka on EC2?
    - **Answer:** I set up a 3-node Kafka cluster on EC2 instances, installed Kafka and ZooKeeper, configured `server.properties` with advertised listeners pointing to the private IPs, and set up replication factor 3. The instances were in private subnets with security groups allowing internal traffic on Kafka's port 9092.
    - **If asked more:** I would explain the challenges — handling ZooKeeper failure, configuring disks (we used EBS gp3 volumes), and tuning OS parameters (vm.swappiness, page cache) for Kafka's disk I/O pattern.
27. What are the challenges of running Kafka on EC2?
    - **Answer:** The main challenges are: (1) disk management — Kafka is I/O intensive, so EBS volume sizing and IOPS provisioning must be right; (2) network latency between brokers affects replication; (3) ZooKeeper adds operational complexity. In cold-chain, a broker once ran out of disk during a retention issue because a consumer wasn't keeping up.
    - **If asked more:** I would explain how I mitigated these by setting disk usage alerts, using multiple EBS volumes with RAID 0, and enabling Kafka's JBOD feature for cheaper storage.
28. How do you manage environment variables in AWS?
    - **Answer:** I use AWS Systems Manager Parameter Store to store environment-specific variables (database URLs, Kafka broker addresses). The Spring Boot application reads these at startup using the AWS SDK. For secrets like passwords, I used AWS Secrets Manager with automatic rotation.
    - **If asked more:** I would explain the difference between Parameter Store (free, up to 10K parameters) and Secrets Manager (paid, automatic rotation), and why I chose Parameter Store for non-sensitive config and Secrets Manager for database credentials.
29. How do you handle secrets?
    - **Answer:** For production secrets (database passwords, API keys), I use AWS Secrets Manager. In CDMS, the Spring Boot application retrieved the database password from Secrets Manager at startup, cached it, and never stored it in code, properties files, or environment variables.
    - **If asked more:** I would explain the rotation process — Secrets Manager can automatically rotate RDS passwords using a Lambda function, and how the application handles stale connections after rotation.
30. What is CI/CD deployment to AWS?
    - **Answer:** CI/CD to AWS means automated build, test, and deployment pipelines using Jenkins. In our setup, Jenkins built the Spring Boot JAR, built a Docker image, published it to Docker Hub, and then deployed it to the EC2 instance by pulling the image and restarting the container.
    - **If asked more:** I would explain the full Jenkins pipeline stages: checkout, compile, test, build Docker image, push to registry, deploy to EC2, run health check, and rollback on failure.
31. How did Jenkins deploy to EC2 in your project?
    - **Answer:** Our Jenkins pipeline had stages for: (1) Maven build and run unit tests, (2) build Docker image with the Spring Boot JAR, (3) push image to Docker Hub, (4) SSH into the EC2 instance, (5) pull the new image, stop the old container, start the new one, and (6) run a health check curl command.
    - **If asked more:** I would explain how Jenkins credentials were stored securely — Docker Hub credentials and EC2 SSH keys were managed as Jenkins credentials and never exposed in the pipeline script.
32. How do you debug AWS deployment failure?
    - **Answer:** I start by checking the Jenkins build output for the specific error — compilation failure, Docker build failure, or deployment script error. Then I check the EC2 instance's CloudWatch logs for the application startup. Common issues: wrong environment variables, missing IAM permissions, or insufficient disk space.
    - **If asked more:** I would share a specific debugging story — a deployment failed because the EC2 instance had run out of disk from old Docker images, so I added a cleanup step to the Jenkins pipeline.
33. What is high availability?
    - **Answer:** High availability means a system stays operational despite component failures. In cold-chain, I achieved HA by deploying Kafka across 3 availability zones, running Spring Boot on 2+ EC2 instances behind a load balancer, and using S3 and RDS multi-AZ for data storage.
    - **If asked more:** I would explain how Kafka's ISR and automatic leader election provided HA for the messaging layer, and how the load balancer's health checks routed traffic away from failed instances.
34. What is fault tolerance?
    - **Answer:** Fault tolerance means a system continues operating correctly after a failure. In cold-chain, Kafka's replication factor of 3 meant the system tolerated a single broker failure without data loss or service interruption. The load balancer's target group marked unhealthy instances and routed traffic to healthy ones.
    - **If asked more:** I would explain how fault tolerance differs from HA — fault tolerance implies zero data loss, while HA implies minimal downtime. Kafka's acks=all with min.insync.replicas=2 provided both.
35. What is horizontal scaling?
    - **Answer:** Horizontal scaling means adding more instances to handle increased load. In cold-chain, when Kafka consumer lag increased, I added more consumer instances (up to the partition count) and scaled the Spring Boot API by increasing the auto-scaling group's desired count.
    - **If asked more:** I would compare horizontal vs vertical scaling — horizontal is more costly but provides elasticity and fault tolerance, while vertical is simpler but has a ceiling.
36. What is vertical scaling?
    - **Answer:** Vertical scaling means increasing the capacity of existing instances (more CPU, memory). I vertically scaled the Kafka broker instances from m5.large (2 vCPU, 8GB) to m5.xlarge (4 vCPU, 16GB) when CPU utilization was consistently above 80%, because adding Kafka brokers doesn't increase per-partition throughput.
    - **If asked more:** I would explain when vertical scaling makes sense — stateful services like databases and Kafka where horizontal scaling is complex — and its limitation: you can only go as high as the largest instance type.
37. What is Databricks?
    - **Answer:** Databricks is a unified analytics platform for big data processing and machine learning. In the cold-chain project, Databricks was used for predictive analytics on historical temperature data to forecast equipment failures and optimize cooling schedules.
    - **If asked more:** I would explain how Databricks consumed data from our Kafka topic (via the Kafka connector) and ran Spark jobs to train predictive models on temperature excursion patterns.
38. How did Databricks help in predictive analytics?
    - **Answer:** Databricks analyzed historical temperature patterns stored in InfluxDB to predict when a cooling unit was likely to fail. The model was trained on temperature fluctuation data and sent predictions back through Kafka to trigger preventive maintenance alerts before an excursion occurred.
    - **If asked more:** I would explain how the predictions reduced unplanned downtime — by alerting the operations team 2-3 hours before a predicted failure, they could service the equipment during planned windows.
39. What is data lake?
    - **Answer:** A data lake stores raw data in its native format, unlike a data warehouse which stores structured data. In the cold-chain project, raw IoT data was stored in S3 as a data lake before being processed by Databricks for analytics. This allowed us to reprocess historical data with new algorithms without data loss.
    - **If asked more:** I would explain the difference between data lake (S3 with Parquet files) and data warehouse (MSSQL tables) — the data lake stored all raw sensor readings, while MSSQL stored aggregated reports.
40. What is cloud cost optimization?
    - **Answer:** Cloud cost optimization means minimizing AWS spend without sacrificing performance. In cold-chain, I optimized costs by: (1) using reserved instances for the Kafka brokers (67% savings vs on-demand), (2) transitioning old S3 reports to Glacier, (3) auto-stopping non-production EC2 instances overnight, and (4) right-sizing underutilized instances.
    - **If asked more:** I would explain how I used Cost Explorer to identify the biggest spend categories, and how setting up budget alerts prevented bill surprises when the team ran extra test environments.

---

## Microservices and System Design Questions

1. What is microservices architecture?
   - **Answer:** Microservices architecture breaks an application into small, independently deployable services that communicate over the network, each owning its own data and logic. In our cold-chain project, separate services for ingestion, alerting, and dashboard could each be deployed independently without affecting others.
   - **If asked more:** I would walk through how our cold-chain system could be refactored into microservices: sensor-ingestion service, alert-engine service, dashboard-api service, each with its own database (InfluxDB for time-series, PostgreSQL for config).
2. Difference between monolith and microservices.
   - **Answer:** A monolith is a single codebase deployed as one unit, while microservices split into separate deployable services. Our CDMS was a monolith with all ETL and reporting in one codebase; microservices would let us scale ETL and reporting independently.
   - **If asked more:** I would discuss tradeoffs: monolith is simpler for small teams but limits independent scaling; microservices add communication and data consistency complexity.
3. Advantages of microservices.
   - **Answer:** Microservices enable independent deployment, scaling, and technology diversity. In our inventory system, if validation logic needed more resources, we could scale only that service instead of the entire application.
   - **If asked more:** I would add that microservices improve fault isolation - one service failing does not bring down the whole system, critical for cold-chain where alerting must stay up even if the dashboard has issues.
4. Disadvantages of microservices.
   - **Answer:** The main drawbacks are distributed complexity, network latency, data consistency challenges, and operational overhead. In the cold-chain project, debugging across Kafka, Spring Boot APIs, and InfluxDB was harder than a monolithic application would have been.
   - **If asked more:** I would explain that microservices need mature DevOps, monitoring, and team coordination. Without proper tooling, tracing a single failed data flow across Lambda, Kafka, and the API layer takes significantly longer than in a monolith.
5. How do services communicate?
   - **Answer:** Services communicate either synchronously via REST/gRPC or asynchronously via message queues like Kafka. In the cold-chain project, IoT data flowed asynchronously through Kafka from Lambda to Spring Boot, while dashboard queries were synchronous REST calls to the API layer.
   - **If asked more:** I would explain the choice criteria: REST for request-response with low latency needs, Kafka for high-throughput decoupled processing where the producer does not need an immediate answer.
6. REST vs messaging.
   - **Answer:** REST is synchronous, simpler, but creates tight coupling. Messaging with Kafka decouples producers from consumers and buffers data. In our cold-chain system, we used both: Kafka for sensor data ingestion and REST for dashboard API queries.
   - **If asked more:** I would compare based on use case - REST for CRUD where the client needs an immediate response, messaging for event-driven workflows like ETL pipelines or real-time data where throughput matters more than immediate reply.
7. What is API gateway?
   - **Answer:** An API gateway is a single entry point that routes requests to backend services, handling authentication, rate limiting, and aggregation. Our Spring Boot API layer acted as a mini-gateway for the React dashboard, consolidating sensor data from InfluxDB and alerts from the alert engine.
   - **If asked more:** I would discuss features like request transformation, circuit breaking, and how gateways simplify client code by hiding service decomposition from frontend consumers.
8. What is service discovery?
   - **Answer:** Service discovery allows services to find each other dynamically without hardcoded addresses. In our EC2-based projects, we used environment variables for service URLs, but in a true microservices setup, tools like Eureka or Kubernetes DNS handle this automatically.
   - **If asked more:** I would explain client-side vs server-side discovery and how load balancers like AWS ALB can act as a discovery mechanism by routing to healthy EC2 instances.
9. What is circuit breaker?
   - **Answer:** Circuit breaker prevents cascading failures by stopping calls to a failing service and failing fast. In our inventory system, if the validation database was slow, a circuit breaker would stop hitting it repeatedly and return a cached response until it recovered.
   - **If asked more:** I would explain the three states (closed, open, half-open) and how Resilience4j configures timeout thresholds and retry policies in Spring Boot.
10. What is retry pattern?
    - **Answer:** Retry pattern automatically re-attempts a failed operation, usually with exponential backoff. In our CDMS ETL pipeline, if a downstream API call failed temporarily, retry with backoff retried up to 3 times before logging a failure.
    - **If asked more:** I would distinguish retry from circuit breaker - retry works for transient failures, while circuit breaker protects against sustained failures. Combining both is recommended.
11. What is timeout?
    - **Answer:** Timeout sets a maximum wait time for an operation, preventing threads from hanging indefinitely. In our cold-chain APIs, we configured connection and read timeouts on REST calls to InfluxDB to ensure the dashboard never waited more than 5 seconds.
    - **If asked more:** I would discuss connection timeout, read timeout, and write timeout, and how they are configured in Spring Boot's RestTemplate or WebClient.
12. What is bulkhead pattern?
    - **Answer:** Bulkhead isolates resources into separate pools so failure in one does not deplete resources for others. We could allocate separate thread pools for real-time sensor processing and dashboard queries, so a slow dashboard query never blocks sensor ingestion.
    - **If asked more:** I would explain thread pool isolation vs semaphore isolation in Resilience4j, and how bulkheads protect critical paths like alerting from less critical paths like reporting.
13. What is rate limiting?
    - **Answer:** Rate limiting controls how many requests a client can make in a time window, protecting backend resources from overload. I understand token bucket and sliding window algorithms - useful for public APIs to prevent abuse.
    - **If asked more:** I would discuss implementing rate limiting at API gateway level using Redis for distributed rate counting, with different limits for different client tiers.
14. What is load balancing?
    - **Answer:** Load balancing distributes traffic across multiple servers to improve availability and throughput. We used AWS Application Load Balancer to distribute requests across our EC2 instances for Spring Boot APIs serving the cold-chain dashboard.
    - **If asked more:** I would explain round-robin vs least-connections algorithms, and how health checks on the ALB automatically removed unhealthy EC2 instances from the target group.
15. What is distributed tracing?
    - **Answer:** Distributed tracing tracks a request as it flows through multiple services, helping debug latency and failures. In the cold-chain project, a correlation ID helped trace a sensor reading across the entire flow from Lambda to Kafka to Spring Boot.
    - **If asked more:** I would explain how tools like Jaeger or Zipkin work with trace IDs and span IDs, and how Spring Cloud Sleuth automatically injects trace IDs into logs and HTTP headers.
16. What is centralized logging?
    - **Answer:** Centralized logging aggregates logs from all services into a single searchable platform. We used CloudWatch Logs to collect and search logs from all Spring Boot EC2 instances without SSHing into each server individually.
    - **If asked more:** I would discuss log levels (ERROR, WARN, INFO, DEBUG), structured logging with JSON format, and how centralized logging correlates errors across services using the trace ID.
17. What is correlation id?
    - **Answer:** A correlation ID is a unique identifier attached to a request as it passes through multiple services, linking all related log entries. In the cold-chain system, each sensor reading carried a UUID from ingestion to dashboard, making end-to-end tracing possible.
    - **If asked more:** I would explain generating correlation IDs at the entry point and propagating them through Kafka message headers and log statements for full visibility.
18. What is eventual consistency?
    - **Answer:** Eventual consistency means that after a write, all replicas will converge to the same state given enough time without new updates. In our inventory system, partner submissions were eventually consistent across reporting views - not instant, but guaranteed within seconds.
    - **If asked more:** I would contrast with strong consistency and discuss tradeoffs: eventual consistency improves availability and performance but requires handling stale reads in application logic.
19. What is distributed transaction?
    - **Answer:** A distributed transaction spans multiple services or databases, requiring coordination for all-or-nothing execution. In our inventory system, partner submissions involved validating records, updating baseline, and calculating inventory - all in MSSQL transactions at the database level.
    - **If asked more:** I would discuss two-phase commit (2PC) drawbacks - latency and reduced availability - and why modern systems prefer eventual consistency with saga patterns.
20. What is saga pattern?
    - **Answer:** Saga is a sequence of local transactions where each step publishes an event to trigger the next, with compensating transactions for rollback. If our inventory had a distributed flow across services, a saga would ensure validation rollback if baseline update failed after validation succeeded.
    - **If asked more:** I would explain choreography vs orchestration sagas - choreography uses events, orchestration uses a coordinator - and when each is appropriate based on complexity needs.
21. What is CQRS?
    - **Answer:** CQRS separates read and write operations into different models, optimizing each for its purpose. In our cold-chain project, writes went to Kafka/InfluxDB for high-throughput ingestion, while reads used optimized InfluxDB queries and Redis caching for fast dashboard rendering.
    - **If asked more:** I would explain that CQRS is useful when read and write workloads differ - our IoT writes were high-volume append-only, while reads were aggregation queries with time range filters.
22. What is event-driven architecture?
    - **Answer:** Event-driven architecture uses events to trigger and communicate between decoupled services. The cold-chain system was fully event-driven: IoT data triggered Lambda, which published to Kafka, which triggered Spring Boot processing, which stored to InfluxDB and triggered dashboard updates.
    - **If asked more:** I would explain event sourcing vs event notification, and how Kafka's log-based approach made our system replayable and auditable.
23. What is idempotency?
    - **Answer:** Idempotency means performing the same operation multiple times produces the same result as doing it once. In our inventory system, if a partner submitted the same serial record twice, the validation engine identified it as duplicate using a unique constraint on the serial number, preventing double-counting.
    - **If asked more:** I would explain idempotency keys - clients send a unique key, the server deduplicates based on that key, storing the result for subsequent identical requests.
24. Why is idempotency important?
    - **Answer:** Idempotency prevents data corruption from duplicate requests caused by network retries or client errors. In the inventory system, without idempotency, partners could accidentally inflate inventory counts, causing reconciliation issues and financial discrepancies.
    - **If asked more:** I would mention that idempotency is critical for payment-like operations and inventory updates where even a single duplicate can have significant business impact.
25. How do you design idempotent APIs?
    - **Answer:** I would use an idempotency key pattern: the client sends a unique key, the server checks if it has already processed that key and returns the cached response. In our inventory API, the batch submission ID served as the idempotency key.
    - **If asked more:** I would discuss storage options for idempotency keys (Redis with TTL or database table), expiration policy, and handling concurrent requests with the same key.
26. How do you handle duplicate requests?
    - **Answer:** For naturally idempotent operations like GET or PUT, no extra handling is needed. For non-idempotent operations like inventory submission, I would add a unique constraint on batch reference and reject duplicates at the database level with appropriate error messaging.
    - **If asked more:** I would add that duplicate detection should have a time window - a duplicate with the same data arriving after a week might be a legitimate new request, so TTL-based deduplication is preferred.
27. How do you handle API versioning?
    - **Answer:** I prefer URL-based versioning (`/api/v1/resource`) because it is explicit and easy to route at the load balancer level. In our internal projects, we did not need aggressive versioning since API consumers were controlled, but for public APIs I would use this approach.
    - **If asked more:** I would discuss backward compatibility: never remove fields clients depend on, add fields as optional, deprecate with a migration timeline documented in Swagger.
28. How do you design pagination?
    - **Answer:** I use cursor-based pagination with a last-seen ID or timestamp for large or real-time datasets because offset-based pagination can be inconsistent when data changes. In the inventory dashboard, offset-based was fine for static partner lists, but for sensor readings, cursor-based prevents duplicates during live streaming.
    - **If asked more:** I would discuss page size limits, total count performance, and caching total counts in Redis instead of counting on every request.
29. How do you design search APIs?
    - **Answer:** I keep search APIs simple with query parameters for filtering, sorting, and pagination. In CDMS, search endpoints accepted date range, partner ID, and product category as query parameters, with WHERE clauses built dynamically using Spring Data JPA Specifications.
    - **If asked more:** I would discuss full-text search with Elasticsearch, indexed columns for filter fields, and preventing SQL injection by using parameterized queries.
30. How do you design notification system?
    - **Answer:** I would design it as an event-driven system: notification events published to a Kafka topic, consumers process them and send via email/SMS/push, with a database storing delivery status. In the cold-chain project, excursion alerts were notifications triggered when temperature exceeded thresholds.
    - **If asked more:** I would discuss delivery guarantees (at-least-once with deduplication), retry mechanisms for failed deliveries, and rate limiting to avoid flooding recipients during multi-sensor excursions.
31. How do you design rate limiter?
    - **Answer:** I would use a sliding window counter stored in Redis, tracking request counts per client per time window using sorted sets. The API gateway would check eligibility before forwarding each request, returning 429 Too Many Requests when the limit is exceeded.
    - **If asked more:** I would explain the token bucket algorithm as an alternative and discuss distributed rate limiting challenges - clock skew and Redis being a single point of failure.
32. How do you design URL shortener?
    - **Answer:** I would use a hash-based approach: generate a unique short key from a hash of the original URL, store the mapping in a database with Redis caching, and return a 302 redirect on lookup.
    - **If asked more:** I would discuss collision handling (if two URLs produce the same hash, append a salt), key length optimization, and analytics tracking for click counts.
33. How do you design distributed cache?
    - **Answer:** I would use Redis with a cache-aside pattern: check Redis first, if not found, fetch from the database and store in Redis with a TTL. In our cold-chain project, we cached frequently accessed sensor metadata and dashboard configurations in Redis to reduce database load.
    - **If asked more:** I would discuss consistent hashing for Redis cluster sharding, handling cache stampede with early expiration and locking, and monitoring cache hit/miss ratios.
34. How do you design chat system?
    - **Answer:** I would design it with WebSocket connections for real-time messaging, Kafka for message persistence and ordering, and a database for message history partitioned by conversation ID.
    - **If asked more:** I would discuss presence detection (heartbeat mechanism), message delivery guarantees, and scaling WebSocket servers using sticky session load balancers.
35. How do you design inventory system?
    - **Answer:** I would design it with a centralized validation engine that receives serial records from partners, validates against a verified baseline, and calculates real-time inventory. This is exactly what we built: MSSQL stored procedures validated 10,000+ records from 200+ partners in under 15 seconds.
    - **If asked more:** I would walk through the exact flow - partner submits batch, validation checks duplicates and invalid serials against baseline, accepted records update inventory, rejected records are flagged for reconciliation.
36. How do you design IoT monitoring system?
    - **Answer:** I would design it with a scalable ingestion pipeline: IoT gateways send MQTT to AWS IoT Core, which triggers Lambda to publish to Kafka, Spring Boot consumers process and store time-series data in InfluxDB, and Grafana/React dashboards query the APIs. This is our cold-chain architecture.
    - **If asked more:** I would explain the reasoning behind each choice - MQTT for lightweight IoT protocol, Kafka for buffering bursts, InfluxDB for time-series optimization.
37. How do you design real-time dashboard?
    - **Answer:** I would design it with polling using React setInterval, backed by optimized InfluxDB queries and Redis caching. In our cold-chain dashboard, we polled the Spring Boot API every 30 seconds for sensor readings, with Redis caching frequently accessed data points.
    - **If asked more:** I would discuss WebSocket vs polling tradeoffs, data aggregation strategies (pre-aggregating at 1-minute intervals), and lazy loading for historical data.
38. How do you handle high throughput ingestion?
    - **Answer:** I would use a message queue like Kafka as a buffer between producers and consumers. In the cold-chain system, hundreds of IoT gateways sent data simultaneously; Kafka absorbed the burst, and consumers processed at their own pace without data loss.
    - **If asked more:** I would discuss partition count planning, consumer group scaling, and monitoring consumer lag to detect processing bottlenecks early.
39. How do you handle data consistency?
    - **Answer:** For same-database consistency, I rely on ACID transactions. For cross-service consistency, I use eventual consistency with idempotent operations. In CDMS, the ETL pipeline used database transactions to ensure partial failures did not leave reporting data inconsistent.
    - **If asked more:** I would discuss CAP theorem tradeoffs - in distributed systems, we often choose availability over strong consistency and design APIs to handle eventual consistency gracefully.
40. How do you handle failure in downstream services?
    - **Answer:** I use timeouts, retries with exponential backoff, circuit breakers, and fallback mechanisms. In the cold-chain system, if InfluxDB was slow, we returned stale data from Redis cache as a fallback instead of showing an error.
    - **If asked more:** I would discuss graceful degradation - the system should partially work even when dependencies fail, and monitoring should alert when fallback paths are activated.
41. How do you choose between sync and async communication?
    - **Answer:** I choose synchronous (REST) when the client needs an immediate response and the operation is quick. I choose asynchronous (Kafka) when high throughput, decoupling, or background processing is needed. In our cold-chain system, sensor ingestion was async, dashboard queries were sync.
    - **If asked more:** I would add that async communication introduces complexity in error handling and tracing, so the decision should consider whether the client can tolerate delayed responses.
42. How do you secure microservices?
    - **Answer:** I secure microservices with JWT-based authentication, role-based access control, HTTPS, API gateway as a security barrier, and input validation at every service boundary. We used Spring Security with JWT tokens for API authentication and configured CORS for the React frontend.
    - **If asked more:** I would discuss OAuth2 for delegated authorization, service-to-service authentication using mutual TLS, and secrets management using environment variables.
43. How do you monitor microservices?
    - **Answer:** I monitor with health check endpoints, metrics (request rate, latency, error rate, resource usage), centralized logging, and distributed tracing. We used Spring Boot Actuator, CloudWatch for metrics and logs, and Grafana dashboards for application-level monitoring.
    - **If asked more:** I would discuss the three pillars of observability - logging, metrics, and tracing - and how each serves a different purpose.
44. What metrics would you track for backend services?
    - **Answer:** I track request rate (throughput), latency percentiles (p50, p95, p99), error rate (4xx/5xx), resource usage (CPU, memory, connections), and business metrics (records processed). In CDMS, we tracked stored procedure execution time as a key metric.
    - **If asked more:** I would explain RED method (Rate, Errors, Duration) for microservices and USE method (Utilization, Saturation, Errors) for resource monitoring.
45. What is SLA?
    - **Answer:** SLA (Service Level Agreement) is a commitment about expected service level, like 99.5% uptime. In our projects, we had informal SLAs - the dashboard had to load within 2 seconds, and sensor data had to be available within 30 seconds of ingestion.
    - **If asked more:** I would discuss how SLAs drive architectural decisions - a 99.9% SLA requires redundancy across AZs, while 99% allows simpler single-region deployment.
46. What is SLO?
    - **Answer:** SLO (Service Level Objective) is an internal target, usually stricter than the SLA. For the cold-chain system, our SLO was 99% of dashboard queries complete within 1 second, giving a buffer before breaching the 2-second SLA.
    - **If asked more:** I would explain how SLOs trigger alerts - if error rate exceeds the SLO for 5 minutes, on-call is paged, catching issues before the SLA is breached.
47. What is error budget?
    - **Answer:** Error budget is the acceptable amount of downtime based on the SLO. If our SLO is 99.9% uptime, the error budget allows 0.1% downtime (about 8.7 hours per year). Teams can deploy confidently within the budget but must prioritize stability when it is depleted.
    - **If asked more:** I would discuss how error budget drives the balance between velocity and reliability - more deployments when the budget is healthy, focus on stability when it is exhausted.
48. How do you design for scalability?
    - **Answer:** I design for scalability by identifying bottlenecks - usually the database or a single-threaded component. For inventory, we optimized queries and indexing rather than adding servers. For cold-chain, Kafka allowed adding more consumers as data volume grew.
    - **If asked more:** I would discuss horizontal vs vertical scaling, stateless API design (store session data in Redis), and database scaling strategies like read replicas and sharding.
49. How do you design for reliability?
    - **Answer:** I design for reliability by eliminating single points of failure, implementing retries and circuit breakers, and monitoring health checks. In the cold-chain project, we ran two Spring Boot instances behind a load balancer so if one failed, the other continued serving.
    - **If asked more:** I would discuss redundancy at every level (multiple AZs, multiple instances, database replication), graceful degradation, and chaos engineering principles.
50. How do you design for maintainability?
    - **Answer:** I design for maintainability with clean, modular code, clear separation of concerns, consistent error handling, and comprehensive logging. In CDMS, I documented optimized stored procedures with comments explaining execution plan changes and why each index was created.
    - **If asked more:** I would discuss coding standards, code reviews, automated tests, and the importance of reducing the time a new team member takes to understand and modify code safely.

---

## REST API Questions

1. What is REST?
   - **Answer:** REST is an architectural style for designing networked applications using stateless HTTP operations on resources identified by URLs. In our cold-chain project, we exposed endpoints like `GET /api/sensors/{id}/readings` to fetch time-series data for a specific sensor.
   - **If asked more:** I would explain the six REST constraints and how statelessness simplified our EC2 deployment - any instance could handle any request without session affinity.
2. What are REST constraints?
   - **Answer:** The six constraints are: client-server separation, statelessness, cacheability, uniform interface (URI-based resource identification), layered system, and optionally code on demand. Our APIs were stateless and used Redis for cacheability.
   - **If asked more:** I would discuss how statelessness impacts scalability - no server-side session needed, any EC2 instance handles any request - and how we used ETags for caching.
3. Difference between REST and SOAP.
   - **Answer:** REST uses JSON/HTTP with resource-based URLs and standard methods, while SOAP is an XML-based protocol with strict schemas and WSDL. I have not used SOAP in my projects; our APIs were always RESTful JSON over HTTP, simpler and faster for web dashboards.
   - **If asked more:** I would explain that SOAP has built-in error handling and WS-Security for enterprise use, while REST is preferred for web/mobile APIs due to simplicity and browser compatibility.
4. Difference between REST and GraphQL.
   - **Answer:** REST has fixed response structures per endpoint, while GraphQL lets clients request exactly the fields they need. Our inventory API used REST because partners needed consistent, predefined reports rather than flexible querying.
   - **If asked more:** I would discuss REST's caching advantage (each URL is cacheable) vs GraphQL's flexibility, and how for real-time IoT data, REST with pagination was sufficient.
5. What are HTTP methods?
   - **Answer:** The main methods are GET (read), POST (create), PUT (full update/replace), PATCH (partial update), DELETE (remove). In our inventory API, we used POST to submit serial records, GET to fetch inventory status, and PUT to update baseline data.
   - **If asked more:** I would explain safe (GET/HEAD/OPTIONS) vs idempotent (GET/PUT/DELETE) methods, and how this affects retry behavior.
6. Difference between PUT and PATCH.
   - **Answer:** PUT replaces the entire resource; PATCH applies a partial update. If updating only a partner's status, PATCH is efficient - send just the status field. PUT requires the full object. In our inventory system, PATCH was useful for updating specific validation flags.
   - **If asked more:** I would discuss that PUT is inherently idempotent, while PATCH requires careful handling to remain idempotent using JSON Patch format.
7. Difference between POST and PUT.
   - **Answer:** POST creates a resource at a server-generated URL (non-idempotent), while PUT creates or replaces at a client-specified URL (idempotent). In CDMS, POST submitted new partner data, PUT updated existing records with known IDs.
   - **If asked more:** I would explain: POST for submissions where the server assigns an ID, PUT for updates where the client knows the resource's exact URL.
8. What is idempotency?
   - **Answer:** Idempotency means making the same request multiple times produces the same result as once. In our inventory submission, batch reference IDs made the endpoint idempotent - retried batches returned existing results instead of processing duplicate records.
   - **If asked more:** I would discuss idempotency keys in headers and how Redis stores key-response mappings with TTL for retries within a time window.
9. Which HTTP methods are idempotent?
   - **Answer:** GET, PUT, DELETE, HEAD, and OPTIONS are idempotent. POST and PATCH are not inherently idempotent. For our critical POST endpoints like inventory submission, we added idempotency-key support separately.
   - **If asked more:** I would explain that DELETE idempotency means the second call returns 404 instead of 200, but the server state remains the same - the resource is still deleted.
10. What are common HTTP status codes?
    - **Answer:** Common codes: 200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 500 Internal Server Error, 503 Service Unavailable. We consistently used these across our APIs.
    - **If asked more:** I would group codes by category - 2xx success, 3xx redirection, 4xx client error, 5xx server error - and explain how consistent status codes make API consumers more predictable.
11. Difference between 400 and 422.
    - **Answer:** 400 means the request is malformed (invalid JSON, missing required fields), while 422 means the format is valid but content violates business rules. In our inventory API, we returned 400 for JSON parse errors and 422 for serial number validation failures.
    - **If asked more:** I would explain the practical distinction: 400 is for syntax-level issues, 422 for semantic-level issues. Proper distinction helps clients show better error messages.
12. Difference between 401 and 403.
    - **Answer:** 401 means the client is not authenticated - needs valid credentials. 403 means the client is authenticated but lacks permission. In our Spring Security setup, 401 was returned for missing/invalid JWT, 403 for insufficient roles.
    - **If asked more:** I would explain that 401 triggers browser login prompts, while 403 does not. Confusing these is a common API design mistake.
13. Difference between 404 and 410.
    - **Answer:** 404 means the resource does not exist currently; 410 means it intentionally existed but was permanently removed. In CDMS, deactivated partners' reports could return 410 to indicate intentional unavailability rather than a missing resource.
    - **If asked more:** I would discuss how many APIs return 404 instead of 403 for hidden resources to avoid revealing existence to unauthorized users.
14. Difference between 500 and 503.
    - **Answer:** 500 is a generic server error; 503 means the server is temporarily unavailable (overloaded or under maintenance). In our cold-chain project, if InfluxDB was down, the API returned 503 to signal temporary unavailability and prompt client retries.
    - **If asked more:** I would explain that 503 should include a Retry-After header, and monitoring should alert on sudden 5xx rate increases.
15. What is request validation?
    - **Answer:** Request validation ensures incoming data meets format, type, and business rules before processing. In our inventory API, we used Spring Boot's `@Valid` with Bean Validation annotations and custom validators for serial number rules - rejecting invalid requests with 400/422 before they reached the service layer.
    - **If asked more:** I would discuss multi-layer validation: schema (field types), format (regex patterns), and business (duplicate checks, status transitions).
16. What is DTO?
    - **Answer:** DTO (Data Transfer Object) decouples the API contract from internal entities. In our projects, DTOs exposed only necessary fields to API consumers, preventing internal entity changes from breaking the API contract and avoiding oversharing sensitive data.
    - **If asked more:** I would explain the security aspect - DTOs prevent exposing password hashes or internal IDs and reduce payload size by including only needed fields.
17. Why not expose entity directly?
    - **Answer:** Exposing JPA entities directly couples the database schema to the API contract - any schema change breaks the API. It also risks exposing sensitive fields and causes lazy loading or infinite serialization issues with bidirectional relationships.
    - **If asked more:** I would add that direct entity exposure can cause performance overhead from loading unnecessary fields and make API evolution harder.
18. What is global exception handling?
    - **Answer:** Global exception handling centralizes error handling across all controllers, returning consistent error responses. In our Spring Boot projects, `@ControllerAdvice` caught validation errors, not-found exceptions, and server errors - all returned in a uniform `{error, message, timestamp}` JSON format.
    - **If asked more:** I would explain our error response structure: HTTP status, machine-readable error code, human-readable message, and timestamp. Consistency helped the React frontend handle errors generically.
19. How do you design error response?
    - **Answer:** I design error responses with a consistent structure: `{"status": 400, "error": "VALIDATION_ERROR", "message": "Serial number must be alphanumeric", "timestamp": "2024-01-15T10:30:00Z"}`. For 422 errors, we included a `details` array with field-level validation information.
    - **If asked more:** I would discuss including a trace ID in error responses for debugging, and the importance of not exposing sensitive information in error messages.
20. What is API versioning?
    - **Answer:** API versioning allows multiple API versions to coexist, giving consumers migration time. I prefer URI versioning (`/api/v1/inventory`) because it is explicit and easy to route. In our internal projects, versioning was not needed due to controlled consumers.
    - **If asked more:** I would compare URI vs header vs query parameter versioning, and recommend URI for simplicity and cacheability.
21. URL versioning vs header versioning.
    - **Answer:** URL versioning places the version in the path (`/api/v1/`), making it visible and easy to route. Header versioning uses a custom Accept header, keeping URLs clean. I prefer URL versioning for simplicity - anyone testing the API can see the version immediately in the URL.
    - **If asked more:** I would discuss that URL versioning can lead to code duplication across versions, while header versioning keeps URL structure clean but requires client library support.
22. What is pagination?
    - **Answer:** Pagination splits large result sets into smaller pages, reducing response size and server load. In the inventory dashboard, partner lists were paginated with page/pageSize parameters, returning metadata like total pages for the UI to render pagination controls.
    - **If asked more:** I would discuss cursor vs offset-based pagination: offset was fine for slow-changing inventory data, but cursor is better for real-time sensor data where new records are constantly added.
23. What is sorting?
    - **Answer:** Sorting allows clients to order results by specified fields. In CDMS reports, the API accepted `sort=partnerName,asc&sort=revenue,desc` for multi-field ordering. We used Spring Data's Sort object with a whitelist to prevent SQL injection via sort fields.
    - **If asked more:** I would discuss sort field whitelisting - never passing user input to ORDER BY without validation, as it risks injection and performance issues from unindexed columns.
24. What is filtering?
    - **Answer:** Filtering narrows results based on criteria like date range, status, or category. In CDMS, partners filtered reports by date range, region, and product line. We used Spring Data JPA Specifications for dynamic WHERE clauses with proper indexes on filtered columns.
    - **If asked more:** I would discuss filter syntax: query parameters (`?status=active`) for simple filters and JSON filter objects for complex nested filters.
25. What is HATEOAS?
    - **Answer:** HATEOAS includes links in API responses to guide clients to related actions. For example, an inventory record response might include links to "update", "delete", and "view-history". We did not use HATEOAS in our projects, but I understand it makes APIs self-documenting.
    - **If asked more:** I would discuss the tradeoff: HATEOAS increases payload size and complexity but enables loose coupling where clients discover actions dynamically.
26. What is content negotiation?
    - **Answer:** Content negotiation lets clients request different response formats using the Accept header. Our APIs primarily returned JSON via `application/json`. Spring Boot handles this automatically based on the Accept header and registered message converters.
    - **If asked more:** I would explain producer vs consumer negotiation and how we used `produces = "application/json"` in controller mappings.
27. What is CORS?
    - **Answer:** CORS is a browser security mechanism controlling which domains can access a web API. In our cold-chain project, the React dashboard ran on a different port than the Spring Boot API, so we configured CORS in Spring Security to allow requests from the dashboard's origin.
    - **If asked more:** I would explain preflight requests (OPTIONS), CORS headers like `Access-Control-Allow-Origin`, and that CORS is only enforced by browsers, not by server-to-server calls or Postman.
28. How do you secure REST APIs?
    - **Answer:** I secure APIs with HTTPS, JWT authentication, role-based access control, input validation, rate limiting, and CORS configuration. We used Spring Security with JWT - users authenticated, received a token, and included it in the Authorization header for subsequent requests.
    - **If asked more:** I would discuss OAuth2 for third-party access and the principle of least privilege - each endpoint validates the authenticated user has permission for the specific action.
29. How do you document REST APIs?
    - **Answer:** I document APIs using OpenAPI/Swagger via springdoc-openapi, which auto-generates interactive documentation from code annotations. This allowed our frontend developers to test endpoints from the browser without needing curl.
    - **If asked more:** I would discuss the importance of keeping docs in sync with code (annotation-based ensures this), adding meaningful descriptions, and including example request/response bodies.
30. What is OpenAPI?
    - **Answer:** OpenAPI is a standard specification for describing REST APIs using JSON/YAML, defining endpoints, parameters, schemas, and authentication. We used springdoc-openapi to auto-generate OpenAPI docs from annotations, producing a Swagger UI for team consumption.
    - **If asked more:** I would explain how OpenAPI enables client SDK generation, automated testing, and API gateway configuration, making it a valuable contract-first approach.
31. How do you test REST APIs?
    - **Answer:** I test at multiple levels: unit tests for service logic, `@WebMvcTest` for controllers with mocked services, `@SpringBootTest` with Testcontainers for integration tests, and Postman collections for end-to-end validation. We used Postman in the cold-chain project before frontend integration.
    - **If asked more:** I would discuss contract testing with Pact, automated smoke tests in the Jenkins pipeline, and performance testing with JMeter to validate response times under load.
32. How do you handle long-running API requests?
    - **Answer:** For operations taking more than a few seconds, I return 202 Accepted with a tracking ID and process asynchronously. The client polls a status endpoint using the tracking ID. In CDMS, large report generation used this async pattern - submit request, get tracking ID, poll for completion.
    - **If asked more:** I would add that webhooks or Server-Sent Events can push completion notifications instead of polling, reducing load on the status endpoint.
33. How do you handle file upload?
    - **Answer:** I use multipart form data with Spring Boot's MultipartFile, validate file type and size, store to filesystem or AWS S3, and return the file URL. In our inventory system, partners uploaded serial record files which were parsed and validated against the baseline.
    - **If asked more:** I would discuss streaming large files to avoid OutOfMemoryError, virus scanning, and using presigned S3 URLs for direct client uploads.
34. How do you handle large API response?
    - **Answer:** I handle large responses with pagination, field selection, gzip compression, and streaming. In the cold-chain project, sensor reading responses were paginated by default, and we enabled gzip compression at the load balancer level for slow network connections.
    - **If asked more:** I would discuss JSON streaming for very large datasets, allowing clients to start processing before the full response is received.
35. How do you handle backward compatibility?
    - **Answer:** I follow the Robustness Principle - be conservative in what you send, liberal in what you accept. I never remove or rename fields, add new fields as optional, and deprecate endpoints gradually with a migration guide in Swagger documentation.
    - **If asked more:** I would discuss what constitutes breaking vs non-breaking changes: adding fields is safe, changing types is breaking, removing endpoints requires versioning.

---

## Redis and Caching Questions

1. What is Redis?
   - **Answer:** Redis is an in-memory key-value store used for caching, session storage, and distributed locking. In the cold-chain project, Redis cached API responses to reduce load on InfluxDB and stored session data so that our stateless Spring Boot instances could share user sessions.
   - **If asked more:** I would explain Redis's single-threaded event loop model and how it achieves sub-millisecond latency by keeping all data in RAM with optional persistence to disk.
2. Why use Redis?
   - **Answer:** Redis is fast (in-memory), supports multiple data structures (strings, hashes, lists, sets), and has built-in features like TTL, pub/sub, and distributed locking. In cold-chain, I used Redis to cache frequently accessed temperature summaries, reducing InfluxDB query load by about 60%.
   - **If asked more:** I would compare Redis with Memcached — Redis's data structure support and persistence options made it more suitable for our session storage and locking needs.
3. What data types does Redis support?
   - **Answer:** Redis supports strings, hashes, lists, sets, sorted sets, bitmaps, hyperloglogs, and streams. In cold-chain, I used strings for cache entries, hashes for session storage (mapping user ID to multiple session fields), and sorted sets for ranking gateways by temperature deviation.
   - **If asked more:** I would explain how I chose the data type based on the access pattern — hashes were perfect for session data because I could read/write individual fields without serializing the entire object.
4. What is cache?
   - **Answer:** A cache stores frequently accessed data in a fast storage layer to reduce latency and backend load. In cold-chain, Redis cached the latest temperature reading from each sensor so the dashboard could display it instantly without querying InfluxDB every time a user loaded the page.
   - **If asked more:** I would explain the cache-aside pattern I used — the application checked Redis first, and only queried InfluxDB on a cache miss, then stored the result in Redis with a TTL.
5. What is cache-aside pattern?
   - **Answer:** In cache-aside (lazy loading), the application checks the cache first; on a miss, it loads data from the database, stores it in the cache, and returns it. I used this for the latest-sensor-reading API — Redis miss meant a query to InfluxDB, after which the result was cached with a 30-second TTL.
   - **If asked more:** I would explain why I chose cache-aside over read-through — it gives the application control over what gets cached and when, and it's simpler to implement without a cache provider abstraction.
6. What is read-through cache?
   - **Answer:** In read-through, the cache itself loads data from the database on a miss, transparent to the application. Redis doesn't natively support read-through; it requires a cache provider like Spring's `@Cacheable` with `CacheLoader`. Spring's cache abstraction handles the miss transparently.
   - **If asked more:** I would explain that read-through is simpler for developers but less flexible — you can't control caching logic (e.g., which fields to cache) without implementing a custom `CacheLoader`.
7. What is write-through cache?
   - **Answer:** In write-through, data is written to both cache and database in the same transaction. I used this pattern for updating gateway status in cold-chain — when a status change was written to MSSQL, the corresponding Redis cache entry was also updated immediately, preventing stale data.
   - **If asked more:** I would discuss the trade-off: write-through ensures cache consistency but adds latency to write operations, so I reserved it for critical data where staleness wasn't acceptable.
8. What is write-behind cache?
   - **Answer:** In write-behind, data is written to cache immediately and asynchronously persisted to the database later. I considered this for sensor reading batching but didn't implement it because losing readings during a Redis crash was unacceptable for the cold-chain compliance requirements.
   - **If asked more:** I would explain the risk — if Redis goes down before the async write completes, data is lost. I used this pattern only for non-critical data like gateway heartbeat timestamps.
9. What is TTL?
   - **Answer:** TTL (Time To Live) is the duration after which a cached entry is automatically evicted. In cold-chain, I set a 30-second TTL for sensor-reading cache entries and a 24-hour TTL for partner configuration data that rarely changed.
   - **If asked more:** I would explain how I chose TTL values — based on how frequently the data changed and how stale the dashboard could be before users noticed.
10. What is cache eviction?
    - **Answer:** Cache eviction removes entries when memory is full. Redis supports multiple eviction policies (LRU, LFU, TTL, etc.). I configured Redis with `allkeys-lru` eviction, which removed the least recently used keys when memory reached the `maxmemory` limit, ensuring the most accessed data stayed cached.
    - **If asked more:** I would explain why I chose LRU over other policies — cold-chain had a predictable access pattern where recently accessed sensors were more likely to be viewed again.
11. What is LRU?
    - **Answer:** LRU (Least Recently Used) evicts entries that haven't been accessed for the longest time. Redis's `allkeys-lru` policy approximates LRU. In cold-chain, this meant that sensors not being actively monitored were evicted first, while the dashboard's frequently viewed sensors stayed cached.
    - **If asked more:** I would explain how Redis's approximate LRU differs from exact LRU — it samples a subset of keys (configurable via maxmemory-samples) rather than tracking all keys, reducing memory overhead.
12. What is cache stampede?
    - **Answer:** A cache stampede occurs when many requests simultaneously miss the cache (e.g., after TTL expiry), all hitting the database at once. In cold-chain, this happened with the gateways list API — every dashboard refresh caused all users to query InfluxDB when the cache expired.
    - **If asked more:** I would explain the impact — InfluxDB CPU spiked to 90% during stampede events, which could delay write operations for new sensor readings.
13. How do you prevent cache stampede?
    - **Answer:** I prevent stampedes by: (1) using mutex locks so only one request rebuilds the cache, (2) randomizing TTLs slightly to prevent simultaneous expiry, and (3) using early recomputation — refreshing the cache before it expires. In cold-chain, I used a Redis lock to serialize cache rebuilds for the gateways list.
    - **If asked more:** I would explain the specific implementation — `jedis.setnx` to acquire the lock, with a 5-second TTL on the lock key to prevent deadlocks if the rebuilding thread crashed.
14. What is cache penetration?
    - **Answer:** Cache penetration happens when requests come for data that doesn't exist in either cache or database, causing repeated database misses. In cold-chain, if a user requested a non-existent sensor ID, Redis would miss, InfluxDB would miss, and the bad request would bypass caching entirely.
    - **If asked more:** I would explain my solution — I cached the null result (or a special "not found" marker) with a short TTL so repeated requests for invalid data didn't hit InfluxDB repeatedly.
15. What is cache avalanche?
    - **Answer:** A cache avalanche occurs when many cached entries expire at the same time, causing a flood of requests to the database. In cold-chain, if all sensor-reading cache entries had the same 30-second TTL and expired together, InfluxDB would get a sudden load spike.
    - **If asked more:** I would explain how I prevented this by adding a random jitter of ±5 seconds to each TTL value, spreading the expiry across a window and preventing simultaneous cache rebuilds.
16. How do you invalidate cache?
    - **Answer:** I invalidate cache by: (1) setting appropriate TTLs for automatic expiry, (2) explicitly deleting keys when data changes (write-through), and (3) using a version prefix in cache keys for bulk invalidation. In cold-chain, when a partner updated gateway configuration, I deleted the corresponding Redis key so the next read would fetch fresh data.
    - **If asked more:** I would explain the version prefix pattern — appending a version number to cache keys (`gateway:123:v2`) so incrementing the version effectively invalidated all old cache entries without iterating through keys.
17. What should not be cached?
    - **Answer:** Don't cache: (1) frequently changing data where staleness is unacceptable, (2) sensitive user data that has compliance requirements, (3) data used only once. In cold-chain, I didn't cache temperature excursion alerts because they needed to be displayed immediately and the alert TTL was unpredictable.
    - **If asked more:** I would explain that caching financial or health-critical data requires careful consideration of how stale values could impact decisions — temperature excursion alerts were real-time critical, so caching added risk without benefit.
18. How do you cache API responses?
    - **Answer:** I cache API responses using Spring Boot's `@Cacheable` annotation with Redis as the backing store. In cold-chain, I annotated the `getLatestTemperature` method with `@Cacheable(value="sensors", key="#sensorId")`, which automatically cached the response and served it on subsequent calls until the TTL expired.
    - **If asked more:** I would explain the cache configuration — I set `spring.cache.redis.time-to-live=30s` and configured the cache key prefix to avoid collisions between different API caches.
19. How do you cache database queries?
    - **Answer:** I cache database query results at the service layer using `@Cacheable` around the repository call. In cold-chain, the gateway configuration query (which rarely changed) was cached with a 1-hour TTL, reducing MSSQL query load for this frequently accessed but static data.
    - **If asked more:** I would explain the risk — if a gateway config changed, the cached version was up to 1 hour stale, so I added a manual cache eviction call in the update endpoint to immediately invalidate the affected entry.
20. How do you use Redis for session storage?
    - **Answer:** I configured Spring Session with Redis as the session store, so user sessions were stored in Redis instead of the application's local memory. This allowed any instance in the auto-scaling group to serve any user request without losing session state, enabling horizontal scaling.
    - **If asked more:** I would explain the configuration — adding `@EnableRedisHttpSession` and the `spring.session.store-type=redis` property, and how session serialization works (Java serialization by default, JSON for cross-language compatibility).
21. How do you use Redis for distributed locking?
    - **Answer:** Distributed locking ensures that only one instance of an application executes a critical section. In the cold-chain project, I used Redis `SETNX` (set if not exists) with a TTL to implement a distributed lock that prevented duplicate processing of the same sensor reading across multiple consumer instances.
    - **If asked more:** I would discuss the Redlock algorithm for stronger guarantees, but explain that for cold-chain's use case, a single Redis node with SETNX was sufficient since occasional lock failures were acceptable due to the idempotent processing logic.
22. What is Redis pub/sub?
    - **Answer:** Redis pub/sub allows messages to be broadcast to multiple subscribers. In cold-chain, I considered using Redis pub/sub for live dashboard updates but chose Kafka instead because Kafka provides persistence and replay — if the dashboard client disconnected, it would miss messages sent via Redis pub/sub.
    - **If asked more:** I would explain the limitation of pub/sub — messages are fire-and-forget, no persistence, no acknowledgment. It's suitable for ephemeral notifications but not for reliable data delivery.
23. What is Redis stream?
    - **Answer:** Redis Streams is an append-only log data structure similar to Kafka topics, with consumer groups and message acknowledgment. I evaluated Redis Streams for cold-chain but chose Kafka because Kafka's partition model provides better throughput and horizontal scaling for our volume.
    - **If asked more:** I would explain when Redis Streams is better than Kafka — simpler setup, lower latency for small volumes (&lt; 1000 msg/sec), and no need for separate broker management.
24. How do you secure Redis?
    - **Answer:** I secure Redis by: (1) not exposing it to the public internet (deployed in private subnet), (2) using a strong password via the `requirepass` config, (3) disabling dangerous commands like `FLUSHALL` and `KEYS` via rename-command in redis.conf, and (4) using TLS encryption for data in transit.
    - **If asked more:** I would explain that Redis's security model is simple — it trusts the network, so network-level security (VPC, security groups) is critical. I never deployed Redis without first changing the default configuration.
25. How did Redis fit in your project?
    - **Answer:** Redis played three roles in the cold-chain project: (1) cache for API responses (latest temperature readings, gateway lists) to reduce InfluxDB load, (2) session storage for Spring Boot instances to enable horizontal scaling, and (3) distributed locking to prevent duplicate processing of sensor events across multiple consumer instances.
    - **If asked more:** I would give the concrete improvement — after adding Redis caching, the dashboard's latest-temperature API latency dropped from 200ms (InfluxDB query) to under 5ms (Redis lookup), and InfluxDB query volume reduced by 60%.

---

## DevOps, Docker, and CI/CD Questions

1. What is Git?
   - **Answer:** Git is a distributed version control system that tracks source code changes, enabling collaboration and rollback. We used Git with GitHub, feature branches for development, and pull requests for code review before merging to main.
   - **If asked more:** I would discuss our branching strategy - simplified Git flow with feature branches off main, short-lived branches for fixes, and release branches for deployment readiness.
2. What is Git branch?
   - **Answer:** A Git branch is a parallel version of the codebase for independent work without affecting the main code. In our workflow, each feature or bug fix had its own branch, and after review, it was merged to main via a pull request.
   - **If asked more:** I would explain how branches enable parallel development and how we resolved merge conflicts by merging the target branch into the feature branch.
3. Difference between merge and rebase.
   - **Answer:** Merge creates a commit combining two branch histories, preserving the complete timeline. Rebase rewrites commit history by applying commits onto another branch linearly. I prefer merge for feature branches to preserve history and rebase for cleaning up local commits before pushing.
   - **If asked more:** I would caution against rebasing shared branches - the rule is never rebase branches others are working on.
4. What is pull request?
   - **Answer:** A pull request (PR) lets developers request review of their branch changes before merging. In our team, every change went through a PR - at least one reviewer checked logic, performance, and testing before merge.
   - **If asked more:** I would discuss PR best practices: keep PRs small and focused, include clear descriptions, ensure CI passes before requesting review.
5. How do you resolve merge conflict?
   - **Answer:** I pull the latest target branch, run `git merge` to see conflicts, then edit conflicting files keeping the correct code. I use VS Code's merge editor for a side-by-side comparison, then run tests after resolution before committing.
   - **If asked more:** I would communicate with the other developer if the conflict involves their code, and ensure the final commit is clean and tested.
6. What is CI/CD?
   - **Answer:** CI/CD automates building, testing, and deploying code changes. In our Jenkins pipeline, every push to main triggered automatic build, test, Docker image creation, and deployment to EC2. CI catches issues early, and CD reduces manual deployment errors.
   - **If asked more:** I would explain the difference: CI focuses on frequent merges and automated testing; CD takes it further by automatically deploying after successful CI.
7. What is Jenkins?
   - **Answer:** Jenkins is an open-source automation server for CI/CD pipelines. We used declarative pipelines in a Jenkinsfile that checked out code, ran Maven build and tests, created a Docker image, pushed to Docker Hub, and deployed to EC2 via SSH.
   - **If asked more:** I would discuss the Jenkins plugin ecosystem, pipeline as code (Jenkinsfile in repo for versioning), and configuring agents for parallel build stages.
8. How do you create Jenkins pipeline?
   - **Answer:** I create declarative pipelines using a Jenkinsfile in the repo root with stages: checkout, build (mvn clean install), test (mvn test), Dockerize (docker build and push), deploy (SSH to EC2, docker pull, docker-compose up). Each stage has clear success/failure conditions.
   - **If asked more:** I would discuss environment-specific configuration using Jenkins credentials for secrets and parameterized builds for manual triggers.
9. What are Jenkins stages?
   - **Answer:** Jenkins stages are logical pipeline steps representing build phases. Our pipeline stages: Checkout, Build, Test (parallel unit and integration), Docker Build, Docker Push, Deploy to Staging, Smoke Test, and Deploy to Production (with manual approval).
   - **If asked more:** I would mention that stages can run in parallel, each with its own agent, and they make pipeline failures visible at a glance.
10. How do you manage Jenkins credentials?
    - **Answer:** We used Jenkins' built-in credential store for Docker Hub credentials, EC2 SSH keys, and database passwords - referenced in the Jenkinsfile using `credentialsId`. This avoided hardcoding secrets in the pipeline code or repository.
    - **If asked more:** I would discuss using environment variables for non-sensitive config and the principle that secrets should never be logged or printed in pipeline output.
11. How do you deploy Spring Boot using Jenkins?
    - **Answer:** Our Jenkins pipeline built the Spring Boot app with Maven, created a Docker image from the JAR, pushed it to Docker Hub, then SSHed into EC2 to pull the new image and restart containers using Docker Compose. Fully automated from commit to deployment.
    - **If asked more:** I would discuss different strategies: Docker Compose for single-host, and later considering AWS ECS for multi-host deployments with load balancing.
12. How did Jenkins reduce deployment time in your project?
    - **Answer:** Jenkins reduced deployment from manual 30-minute SSH-and-copy sessions to automated 5-7 minute pipelines. Before Jenkins, developers manually copied JARs to EC2 and restarted services. Now a webhook triggers the full pipeline, eliminating human error and reducing downtime.
    - **If asked more:** I would explain specific time savings: manual deployment took 30 minutes with config mistake risks; automated pipeline took 7 minutes with consistent, repeatable steps.
13. What is Docker?
    - **Answer:** Docker packages applications with all dependencies into lightweight, portable containers. In our projects, each service (Spring Boot API, Kafka, InfluxDB, Redis) ran in its own Docker container, ensuring consistent environments across dev, staging, and production on EC2.
    - **If asked more:** I would explain the difference from VMs (full OS per instance) vs containers (shared OS kernel, isolated processes), and how Docker improved our EC2 resource utilization.
14. Why use Docker?
    - **Answer:** Docker eliminates environment inconsistencies by packaging the application with all dependencies. In the cold-chain project, Docker ensured Spring Boot, Kafka, and InfluxDB versions were identical everywhere, and container startup and recovery was fast.
    - **If asked more:** I would discuss isolation benefits - each container has its own filesystem, network, and process space - and how Docker Compose simplified local development with one command for all services.
15. What is Docker image?
    - **Answer:** A Docker image is a read-only template with instructions for creating a container, containing the application code, runtime, libraries, and configuration. We built images from a Dockerfile using `docker build` and stored them on Docker Hub for EC2 distribution.
    - **If asked more:** I would explain image layering - each Dockerfile instruction creates a layer, and layers are cached to speed up subsequent builds.
16. What is Docker container?
    - **Answer:** A Docker container is a running instance of a Docker image - an isolated process with its own filesystem and network. Each container (Spring Boot, Kafka, Redis) shared the EC2 host's kernel but had isolated environments, making them portable across any Docker host.
    - **If asked more:** I would discuss container lifecycle - create, start, stop, restart, remove - and Docker's health check feature for automatic restart on failure.
17. What is Dockerfile?
    - **Answer:** A Dockerfile is a script with instructions to build a Docker image. Our Spring Boot Dockerfile used multi-stage build: first stage compiled code with Maven, second stage used a slim OpenJDK image to run the JAR, keeping the image around 150MB.
    - **If asked more:** I would walk through our actual Dockerfile: FROM maven:3.8 AS build, FROM openjdk:17-jre-slim, COPY --from=build, ENTRYPOINT `java -jar app.jar`.
18. What is Docker Compose?
    - **Answer:** Docker Compose defines multi-container applications using a YAML file. In the cold-chain project, our `docker-compose.yml` defined Spring Boot API, Kafka, Zookeeper, Redis, and InfluxDB - all starting in order with one `docker-compose up` command.
    - **If asked more:** I would explain service dependencies (depends_on), environment variables, port mappings, and volume mounts for persistent data in the compose file.
19. Difference between image and container.
    - **Answer:** An image is a static, read-only template (like a class), while a container is a running instance (like an object). Images are stored in registries and versioned; containers are ephemeral, can be started, stopped, and deleted without affecting the image.
    - **If asked more:** I would use the analogy: image is like a recipe, container is like the cooked meal. Multiple containers can run from the same image, each with its own state.
20. What is multi-stage Docker build?
    - **Answer:** Multi-stage build uses multiple FROM statements, copying artifacts from intermediate stages into the final stage. Our Spring Boot Dockerfile used this: build compiled with Maven, runtime only had the JAR and JDK, reducing the image from 700MB to ~150MB.
    - **If asked more:** I would discuss how multi-stage builds improve security by excluding build tools from the final image, reducing the attack surface in production.
21. How do you reduce Docker image size?
    - **Answer:** I use multi-stage builds, choose slim base images (openjdk:17-jre-slim), clean package manager caches, and minimize layers. Our Spring Boot image went from 700MB to 150MB. Using `.dockerignore` excludes unnecessary files like logs and .git.
    - **If asked more:** I would mention tools like `dive` to analyze image layer contents and identify space wastage.
22. How do you pass environment variables to Docker?
    - **Answer:** We passed environment variables through Docker Compose's `environment` section or `.env` files. For production, Jenkins injected them at container runtime. Spring Boot's properties were overridden using environment variables like `SPRING_DATASOURCE_URL` mapped in the compose file.
    - **If asked more:** I would discuss the 12-factor app methodology - configuration in environment variables, not code - and separate .env files for dev/staging/production.
23. How do you debug container startup failure?
    - **Answer:** I check `docker logs <container>` for startup errors, verify environment variables, inspect the Dockerfile for entry point issues, and check if required services are available. Spring Boot startup failures were often due to database connection issues or missing env vars.
    - **If asked more:** I would also use `docker exec` for diagnostics, check resource limits that prevent startup, and verify port availability on the host.
24. What is logging in Docker?
    - **Answer:** Docker containers output logs to stdout/stderr, collected by the logging driver. We used the JSON-file driver locally and CloudWatch agent for centralized log collection on EC2. Each container's logs were accessible through `docker logs` and aggregated in CloudWatch Logs.
    - **If asked more:** I would discuss different logging drivers (syslog, fluentd, awslogs) and structured JSON logging for better searchability.
25. What is health check?
    - **Answer:** A health check verifies that a container is running correctly beyond just the process being alive. In our Spring Boot containers, we used Actuator's `/actuator/health` endpoint. Docker restarted containers that failed health checks, and the load balancer removed them from the target group.
    - **If asked more:** I would explain liveness vs readiness probes and how Spring Boot separates these with dedicated Actuator endpoints.
26. What is deployment rollback?
    - **Answer:** Deployment rollback reverts to a previous stable version when a deployment causes issues. We kept the last known-good Docker image tag, and rollback meant re-tagging the previous image and redeploying. We documented and tested the rollback procedure.
    - **If asked more:** I would discuss the importance of backward-compatible database changes - code rollback is safe only if DB schema changes are also backward-compatible.
27. What is blue-green deployment?
    - **Answer:** Blue-green deployment runs two identical environments (blue=current, green=new) and switches traffic after testing. We did not implement this on EC2 due to cost, but for zero-downtime, I would use AWS ALB target group switching to route traffic between environments.
    - **If asked more:** I would explain that blue-green reduces downtime to traffic switch time and enables quick rollback, but requires double infrastructure during transition.
28. What is canary deployment?
    - **Answer:** Canary deployment routes a small percentage of traffic to the new version while keeping most on the old version, monitoring before full rollout. On EC2, we could use ALB weighted target groups - send 10% traffic to new version, monitor for 15 minutes, then gradually increase to 100%.
    - **If asked more:** I would discuss metrics to monitor during canary (error rate, latency, business metrics) and automated rollback triggers.
29. What is infrastructure as code?
    - **Answer:** Infrastructure as Code (IaC) manages infrastructure through configuration files instead of manual setup. We used Docker Compose as basic IaC for container configuration. For advanced needs, Terraform or CloudFormation define EC2 instances, load balancers, and security groups as code.
    - **If asked more:** I would explain how IaC enables version-controlled, repeatable, auditable infrastructure and reduces configuration drift between environments.
30. What are common deployment issues?
    - **Answer:** Common issues: environment-specific config mistakes, database schema mismatch, dependency version conflicts, resource exhaustion (disk full, memory leak), and health check failures. We once had a Spring Boot container fail on EC2 because of timezone mismatch causing JWT validation errors.
    - **If asked more:** I would discuss mitigations: comprehensive smoke tests, canary deployments for risky changes, monitoring dashboards for early detection, and runbooks for quick recovery.

---

## Testing Questions

1. What is unit testing?
   - **Answer:** Unit testing tests individual components in isolation, mocking external dependencies. In our inventory project, I wrote JUnit + Mockito tests for the validation service - mocking the repository and testing validation logic with various serial number formats.
   - **If asked more:** I would explain the FIRST principles (Fast, Isolated, Repeatable, Self-validating, Timely) and the test pyramid: unit tests at the base (fast, numerous), integration above, and end-to-end at the top (slow, few).
2. What is integration testing?
   - **Answer:** Integration testing verifies that components work together correctly, often involving real databases or message brokers. In our cold-chain project, we used `@SpringBootTest` with Testcontainers to test the full Kafka consumer to InfluxDB flow.
   - **If asked more:** I would discuss how integration tests catch configuration issues that unit tests miss, and how we balanced unit vs integration coverage.
3. Difference between unit and integration test.
   - **Answer:** Unit tests test a single component in isolation (mocked dependencies), while integration tests test multiple components together (real dependencies). Our inventory validation service had unit tests for rules and integration tests for the full submission flow from controller to database.
   - **If asked more:** I would explain the test pyramid ratio: roughly 70% unit, 20% integration, 10% end-to-end.
4. What is JUnit?
   - **Answer:** JUnit is the standard testing framework for Java with annotations like `@Test` and assertions. We used JUnit 5 (Jupiter) for all tests with Maven Surefire plugin running them in the CI pipeline.
   - **If asked more:** I would discuss JUnit 5 features: parameterized tests for multiple inputs, nested tests for organization, and dynamic tests for data-driven scenarios.
5. What is Mockito?
   - **Answer:** Mockito creates mock objects for testing components in isolation. In our service-layer tests, we used `when(repository.findById(any())).thenReturn(Optional.of(entity))` to test business logic without needing a database.
   - **If asked more:** I would discuss verify for checking method calls, argument matchers, spy for partial mocking, and `@InjectMocks` for automatic mock injection.
6. What is mock?
   - **Answer:** A mock is a fake object that mimics a real dependency with expectations set during the test. In testing our inventory service, we mocked the repository to return predefined data, making tests fast, isolated, and deterministic.
   - **If asked more:** I would distinguish mocks from stubs: stubs provide predefined answers, mocks additionally verify that methods were called with expected parameters.
7. What is stub?
   - **Answer:** A stub returns predefined responses to method calls, making the test environment predictable. In our CDMS tests, we stubbed stored procedure calls to return known result sets, testing reporting logic without running the actual procedure.
   - **If asked more:** I would explain the difference: stubs focus on providing data for state testing, mocks focus on verifying interaction behavior.
8. Difference between mock and spy.
   - **Answer:** A mock creates a completely fake object with no real behavior, while a spy wraps a real object and allows overriding specific methods. We used mocks for repositories (completely fake) and spies when we needed some real behavior but wanted to stub others.
   - **If asked more:** I would recommend mocks by default and spies only when necessary, as spies can have side effects from calling real methods.
9. What is `@Mock`?
   - **Answer:** `@Mock` is a Mockito annotation that creates and injects a mock for the annotated field. We annotated repository dependencies with `@Mock` in test classes, using `MockitoAnnotations.openMocks(this)` or `@ExtendWith(MockitoExtension.class)` for initialization.
   - **If asked more:** I would explain how this reduces boilerplate compared to `Mockito.mock()` calls.
10. What is `@InjectMocks`?
    - **Answer:** `@InjectMocks` creates an instance of the annotated class and injects mocks into its dependencies. In our service tests, the service class had `@InjectMocks`, and Mockito automatically injected the `@Mock` repositories, eliminating manual constructor calls.
    - **If asked more:** I would discuss how Mockito handles injection - constructor preferred, then setter, then field - and the importance of a single constructor.
11. What is `@MockBean`?
    - **Answer:** `@MockBean` adds a mock to the Spring application context, replacing any existing bean. In our `@WebMvcTest` controller tests, we used `@MockBean` to mock service layer beans while testing only the controller layer.
    - **If asked more:** I would explain the difference from `@Mock`: `@MockBean` affects the Spring context (slower but necessary for Spring slice tests), while `@Mock` is for plain unit tests.
12. What is `@SpringBootTest`?
    - **Answer:** `@SpringBootTest` loads the full Spring Boot application context for integration testing. In our cold-chain project, we used it with Testcontainers to test the full flow from controller to Kafka to InfluxDB, verifying Spring bean wiring and configuration.
    - **If asked more:** I would discuss `webEnvironment = RANDOM_PORT` for real HTTP testing with TestRestTemplate and `@ActiveProfiles("test")` for test-specific configuration.
13. What is `@WebMvcTest`?
    - **Answer:** `@WebMvcTest` loads only the web layer for focused controller testing. We used it with `@MockBean` for services, testing endpoint mappings, request validation, status codes, and error responses without loading the full context.
    - **If asked more:** I would explain how it is faster than `@SpringBootTest` and how we used MockMvc for HTTP request assertions.
14. What is Testcontainers?
    - **Answer:** Testcontainers provides disposable Docker containers for integration testing. In our cold-chain project, we used it to spin up real Kafka and InfluxDB containers during tests, ensuring tests ran against the same versions as production.
    - **If asked more:** I would discuss how Testcontainers improves reliability over in-memory alternatives by using real dependencies, with `@Container` and `@Testcontainers` annotations for lifecycle management.
15. Why use Testcontainers?
    - **Answer:** Testcontainers ensures tests run against real databases, not in-memory simulations that behave differently. H2 did not support all MSSQL features our stored procedures used, so Testcontainers with a real MSSQL container caught compatibility issues earlier.
    - **If asked more:** I would discuss the tradeoff: slower (container startup) but more reliable. We ran them in CI only, not during local development for every change.
16. How do you test repository layer?
    - **Answer:** I use `@DataJpaTest` which loads only JPA beans and uses an embedded or Testcontainers database. In CDMS, we tested custom queries, pagination, and sorting - verifying derived queries generated correct SQL and returned expected results.
    - **If asked more:** I would discuss testing native queries and `@Query` annotations with parameter binding and projection interfaces.
17. How do you test service layer?
    - **Answer:** I use JUnit and Mockito, mocking repository dependencies and verifying business logic. In the inventory service, we mocked the repository, then tested validation rules, error handling, and edge cases like duplicate serials and invalid formats.
    - **If asked more:** I would discuss testing happy path, validation failures, resource not found, concurrent modifications, and keeping tests independent and fast.
18. How do you test controller layer?
    - **Answer:** I use `@WebMvcTest` with MockMvc, mocking service layer beans. In our cold-chain API tests, we verified GET endpoints returned correct JSON, POST with invalid data returned 400 with validation errors, and endpoints required proper authentication.
    - **If asked more:** I would discuss testing response status codes, headers, JSON structure, and security filters (JWT, role-based access).
19. How do you test Kafka consumer?
    - **Answer:** I use `@EmbeddedKafka` from Spring Kafka test for lightweight tests or Testcontainers with a real Kafka container. In our cold-chain project, we published test messages, verified the consumer processed them, and asserted expected data was stored in InfluxDB.
    - **If asked more:** I would discuss testing deserialization errors, poison pills, offset commit behavior, and consumer retry logic.
20. How do you test scheduled jobs?
    - **Answer:** I extract the job logic into a testable service and test that directly, plus an integration test that triggers the scheduler and verifies the outcome. In our inventory system, the reconciliation job was tested by calling the reconciliation service directly with test data.
    - **If asked more:** I would discuss avoiding testing the scheduler framework itself and focusing on the logic being idempotent and handling failures gracefully.
21. How do you test security rules?
    - **Answer:** I use `@WithMockUser` for role-based access and manually set authentication for token-based tests. We tested that unauthenticated requests returned 401, wrong roles got 403, and correct roles could access endpoints.
    - **If asked more:** I would discuss testing method-level security (`@PreAuthorize`) and CSRF protection.
22. What is code coverage?
    - **Answer:** Code coverage measures the percentage of code executed by tests. We used JaCoCo with Maven, targeting 70-80% coverage for service layers with clear exclusion rules for DTOs, configurations, and generated code.
    - **If asked more:** I would discuss that high coverage does not guarantee good tests - we focused on branch coverage for business logic rather than line coverage numbers.
23. Is 100% coverage always good?
    - **Answer:** No. 100% coverage can be misleading if tests only check simple paths with no meaningful assertions. We focused coverage on business-critical code - validation logic, calculations, error handling - not on boilerplate getters or configuration classes.
    - **If asked more:** I would explain that mutation testing is a stronger measure than line coverage: it checks if tests actually catch bugs when the code is mutated.
24. What is regression testing?
    - **Answer:** Regression testing ensures new changes do not break existing functionality. In our CI pipeline, the full test suite ran on every pull request - unit, integration, and smoke tests - catching regressions before they reached production.
    - **If asked more:** I would discuss how automated regression testing reduces the fear of making changes, especially for critical paths like inventory validation where a regression could cause financial discrepancies.
25. What is smoke testing?
    - **Answer:** Smoke testing is a quick check that the application starts and core functionality works. In our Jenkins pipeline, after deploying to staging, smoke tests checked the health endpoint returned 200, a simple API call worked, and the dashboard loaded - all within 60 seconds.
    - **If asked more:** I would explain that smoke tests catch obvious failures early (wrong config, missing dependencies) and prevent wasting time on deeper testing of a broken system.
26. What is API testing?
    - **Answer:** API testing validates REST endpoints by sending HTTP requests and verifying status codes, headers, response body, and performance. We used Postman for manual testing and automated API tests in the CI pipeline using Spring Boot's TestRestTemplate.
    - **If asked more:** I would discuss testing valid requests (200), invalid input (400), unauthorized (401/403), not found (404), and edge cases like empty responses and large payloads.
27. What is Postman?
    - **Answer:** Postman is a tool for developing and testing APIs through a graphical interface. We created Postman collections for each API module with environment variables, pre-request scripts for authentication, and test scripts for assertions.
    - **If asked more:** I would discuss how collections were version-controlled in the repo and used for QA testing and CI via Newman.
28. How do you test error cases?
    - **Answer:** I intentionally send invalid inputs and assert the correct error response. In the inventory API, we sent missing fields, invalid formats, and duplicate entries - verifying 400/422 status codes and checking error messages were actionable for partners.
    - **If asked more:** I would discuss testing both client errors (4xx) and server errors (5xx), including database failures (mocked exceptions) and timeout scenarios.
29. How do you test performance changes?
    - **Answer:** I run the same workload before and after changes, measuring response times, throughput, and resource usage. In CDMS optimization, I ran stored procedures with realistic data, recorded execution plans, measured elapsed time - from 5 hours to under 12 minutes.
    - **If asked more:** I would discuss consistent test conditions (same data, server load, warm caches) and tools like JMeter for load testing.
30. How do you validate database optimization results?
    - **Answer:** I compare execution plans before and after changes, checking index usage (seek vs scan), logical reads, and actual execution time with realistic data. For CDMS, I captured plans, identified expensive operators, applied changes, and verified index seeks replaced scans.
    - **If asked more:** I would discuss testing with production-like data volumes, validating correctness by comparing output, and monitoring optimized procedures in production for at least one reporting cycle.

---

## React and Frontend Questions

1. What is React?
   - **Answer:** React is a JavaScript library for building UIs with a component-based architecture and declarative rendering. In our cold-chain project, React powered the real-time monitoring dashboard with reusable components for sensor cards, alert panels, and time-series charts.
   - **If asked more:** I would discuss how React's virtual DOM improved dashboard performance by minimizing actual DOM updates, critical for rendering frequently updating sensor data without UI jank.
2. What are components?
   - **Answer:** Components are reusable, self-contained UI pieces that manage their own state and rendering. In our cold-chain dashboard, we had `SensorCard` (temperature/humidity), `AlertBanner` (excursion warnings), and `TimeSeriesChart` (historical data).
   - **If asked more:** I would explain the component hierarchy: `Dashboard` composed `Sidebar`, `Header`, `SensorGrid` with individual `SensorCard` components receiving data via props.
3. What is JSX?
   - **Answer:** JSX is a syntax extension for JavaScript that looks like HTML but compiles to React elements. In our project, `<SensorCard sensor={sensor} onSelect={handleSelect} />` was cleaner than nested `React.createElement` calls.
   - **If asked more:** I would discuss JSX expressions using `{}` for JavaScript and how Babel transpiles JSX during build.
4. What is state?
   - **Answer:** State is data that changes over time within a component, triggering re-renders when updated. In our cold-chain dashboard, `sensorData` state held the latest readings from the API, and updating it automatically re-rendered the sensor cards.
   - **If asked more:** I would discuss component state (useState) vs global state (Context) and the principle of state lifting to the closest common ancestor.
5. What are props?
   - **Answer:** Props are read-only inputs passed from parent to child components, like function arguments. In our dashboard, `SensorGrid` passed sensor objects as props to each `SensorCard`: `<SensorCard name={s.name} value={s.temperature} />`.
   - **If asked more:** I would discuss prop drilling, TypeScript interfaces for prop validation, and default props for optional values.
6. Difference between state and props.
   - **Answer:** State is mutable data managed within a component; props are immutable data passed from a parent. In our dashboard, sensor data came as props from the parent, while UI state like "which panel is expanded" was local component state.
   - **If asked more:** I would explain unidirectional data flow: state in parents flows down as props, children communicate up via callback props.
7. What are hooks?
   - **Answer:** Hooks let functional components use state and lifecycle features. We used `useState` for UI state, `useEffect` for API calls, `useMemo` for expensive computations, and `useCallback` for stable callback references.
   - **If asked more:** I would mention the rules of hooks: only call at top level, only from React functions. We followed these strictly.
8. What is `useState`?
   - **Answer:** `useState` returns a state variable and a setter that triggers re-renders. In our dashboard, `const [selectedSensor, setSelectedSensor] = useState(null)` managed which sensor detail panel was open.
   - **If asked more:** I would discuss initial state, async updates, and the functional form `setCount(prev => prev + 1)` for state depending on previous value.
9. What is `useEffect`?
   - **Answer:** `useEffect` runs side effects after rendering - API calls, subscriptions, DOM updates. In our cold-chain dashboard, it fetched sensor data on mount and set up polling for live updates.
   - **If asked more:** I would discuss cleanup functions, the dependency array (empty = run once, omitted = run every render), and avoiding infinite loops.
10. What is `useMemo`?
    - **Answer:** `useMemo` memoizes an expensive computation result, recalculating only when dependencies change. In our inventory dashboard, we used it to compute aggregated statistics from partner data, avoiding recalculation on every render.
    - **If asked more:** I would distinguish `useMemo` (memoizes a value) from `useCallback` (memoizes a function).
11. What is `useCallback`?
    - **Answer:** `useCallback` memoizes a function reference, preventing child re-renders when the reference changes unnecessarily. We wrapped `handleSensorSelect` in `useCallback` so child `SensorCard` components did not re-render for unrelated parent state changes.
    - **If asked more:** I would explain that `useCallback(fn, deps)` is essentially `useMemo(() => fn, deps)` and is most useful with optimized child components.
12. What is controlled component?
    - **Answer:** A controlled component has its value controlled by React state: `<input value={filterText} onChange={(e) => setFilterText(e.target.value)} />`. React state is the single source of truth.
    - **If asked more:** I would discuss the difference from uncontrolled components (DOM handles its own state via refs) and why controlled is preferred for validation and dynamic UIs.
13. What is uncontrolled component?
    - **Answer:** An uncontrolled component manages its own state via the DOM, accessed through refs. While we generally used controlled components, uncontrolled can be simpler for forms that only need values on submit.
    - **If asked more:** I would explain that uncontrolled components use `useRef` and are less testable since state is not in React.
14. What is conditional rendering?
    - **Answer:** Conditional rendering shows different UI based on conditions using ternaries, `&&`, or if-else in JSX. In our dashboard, `{sensor.status === 'alert' && <AlertIcon />}` showed alert icons only for abnormal readings.
    - **If asked more:** I would discuss ternary for if-else, `&&` for simple show/hide, and early returns for complex conditions.
15. What is list rendering?
    - **Answer:** List rendering uses `Array.map()` to transform data arrays into React elements. In our dashboard: `{partners.map(p => <PartnerRow key={p.id} partner={p} />)}`.
    - **If asked more:** I would discuss empty list handling ("no data" message), loading states, and rendering paginated lists efficiently.
16. Why is key needed in list rendering?
    - **Answer:** Keys help React identify which items changed, were added, or removed, enabling efficient DOM updates. We always used unique IDs as keys - never array indices, which cause incorrect re-rendering when list order changes.
    - **If asked more:** I would explain that keys must be stable, unique, and predictable. Index as key is acceptable only for static, non-reordered lists.
17. What is React Router?
    - **Answer:** React Router enables client-side routing for SPAs without full page reloads. In our dashboard, it managed routes like `/dashboard`, `/alerts`, `/reports`, and `/settings` with lazy loading.
    - **If asked more:** I would discuss BrowserRouter vs HashRouter, protected routes with auth checks, and route parameters for sensor details.
18. What is RBAC in frontend?
    - **Answer:** RBAC restricts UI elements based on user role. In our dashboard, admins saw "Settings" and "Delete" buttons, while viewers saw read-only dashboards. Role info came from JWT claims.
    - **If asked more:** I would emphasize that frontend RBAC is UX convenience, not security - all sensitive operations must be enforced on the backend.
19. How do you protect routes in React?
    - **Answer:** We created a `ProtectedRoute` wrapper that checked auth state from the JWT in localStorage. If authenticated, the route rendered; otherwise, it redirected to login using React Router's `Navigate`. Role-based checks were added for admin-only routes.
    - **If asked more:** I would discuss handling token expiry by redirecting to login with a session-expired message.
20. How do you call APIs from React?
    - **Answer:** We used the Fetch API with async/await in `useEffect`, with a centralized API utility handling base URL, JWT headers, and error handling. In the cold-chain dashboard, `useEffect` polled every 30 seconds and updated state.
    - **If asked more:** I would discuss Axios as an alternative with interceptors for token refresh and request cancellation for unmounted components.
21. How do you handle loading state?
    - **Answer:** We managed loading with a boolean: `const [loading, setLoading] = useState(true)`, set to false after the API response. The UI showed a spinner while loading, replaced by data or an error message.
    - **If asked more:** I would discuss skeleton screens for better UX and handling loading for individual components vs the whole page.
22. How do you handle errors in UI?
    - **Answer:** We displayed error states with an `ErrorBanner` showing a descriptive message and optional retry button. API errors were caught in the fetch, and the error message was extracted from the response body. Network errors showed "connection lost" with auto-retry.
    - **If asked more:** I would discuss error boundaries to prevent the whole dashboard from crashing due to one failed component.
23. How do you optimize React performance?
    - **Answer:** We optimized with `React.memo` for pure components, `useMemo`/`useCallback` for expensive work, code splitting with lazy loading, and keeping state as local as possible to minimize re-renders.
    - **If asked more:** I would discuss React DevTools Profiler, virtualization with react-window for large lists, and debouncing rapid state updates.
24. What is lazy loading?
    - **Answer:** Lazy loading defers component loading until needed, reducing initial bundle size. In our dashboard, `ReportsPage` was lazy-loaded: `const ReportsPage = React.lazy(() => import('./ReportsPage'))`, loading only when the user navigated to reports.
    - **If asked more:** I would discuss combining lazy loading with Suspense for loading states and Webpack chunking.
25. What is code splitting?
    - **Answer:** Code splitting breaks the JS bundle into smaller chunks loaded on demand, improving initial page load. Our dashboard used route-based splitting: each route's component was in a separate chunk, loaded only when visited. Initial load was ~200KB instead of 1MB.
    - **If asked more:** I would discuss Webpack dynamic imports, analyzing bundle size with source-map-explorer, and splitting vendor libraries for caching.
26. How did you build dashboards in React?
    - **Answer:** We built dashboards with a modular component architecture: `DashboardLayout` with configurable grid areas, reusable widgets (`SensorWidget`, `AlertWidget`, `ChartWidget`), and a data layer polling the Spring Boot API. Each widget was independent.
    - **If asked more:** I would discuss the customizable grid layout using react-grid-layout for drag-and-drop widget positioning and per-widget data fetching to isolate loading/error states.
27. What are customizable dashboard grids?
    - **Answer:** Customizable grids let users rearrange widgets via drag-and-drop, resize them, and save layout preferences. We used react-grid-layout, storing the layout JSON in localStorage or sending it to the backend for persistence.
    - **If asked more:** I would discuss responsive breakpoints for different screen sizes and how layout preferences persisted across sessions.
28. What is Three.js?
    - **Answer:** Three.js is a 3D JavaScript library using WebGL for rendering 3D scenes in browsers. In our cold-chain project, we used it to create a 3D digital twin of the warehouse with sensor locations and real-time temperature hotspots.
    - **If asked more:** I would discuss basic Three.js concepts: scene, camera, renderer, and mapping sensor coordinates to 3D positions.
29. How did you use Three.js for digital twin visualization?
    - **Answer:** We built a 3D warehouse model with floor plans, shelving units, and sensor spheres at physical locations. Sensor readings were color-mapped (green=normal, yellow=warning, red=alarm), and clicking a sensor sphere showed its real-time data panel.
    - **If asked more:** I would discuss loading geometry from JSON, updating colors in real-time using `requestAnimationFrame`, and the performance challenge of rendering hundreds of sensors.
30. How do you show live sensor hotspots in UI?
    - **Answer:** Live hotspots were color-coded overlays on the 3D model. Each sensor sphere color updated based on latest temperature (blue=cold, red=hot), and a heatmap effect interpolated colors between sensor positions.
    - **If asked more:** I would discuss the polling mechanism in `useEffect` with `setInterval`, updating Three.js object material colors, and the interpolation algorithm for the heatmap effect.

---

## Scenario-Based Questions

1. A stored procedure suddenly becomes slow in production. How do you debug it?
   - **Answer:** I would check if the execution plan changed - parameter sniffing or stale statistics can cause the optimizer to choose a different plan. In CDMS, a procedure slowed because statistics were stale after a large data load; updating statistics restored performance.
   - **If asked more:** I would examine wait stats, compare actual vs estimated rows in the execution plan, test with OPTION (RECOMPILE) to rule out parameter sniffing, then decide between index fixes or plan guide.
2. A query uses an index in testing but not in production. What could be the reason?
   - **Answer:** Data distribution differences between environments - production has more data with different cardinality, making the optimizer choose a scan over a seek. In CDMS, a query used an index in dev but scanned in production because the indexed column had many NULL values in production.
   - **If asked more:** I would compare statistics between environments, check parameter sniffing, examine actual execution plans from production, and consider index maintenance or query hints after thorough testing.
3. A database job takes 5 hours. How would you reduce it?
   - **Answer:** I would analyze the execution plan to find the most expensive operations and target them with index improvements or query rewrites. In CDMS, I reduced a job from 5 hours to under 12 minutes by identifying missing indexes, rewriting correlated subqueries as joins, and breaking the procedure into batch operations.
   - **If asked more:** I would discuss batch processing (10K records at a time), partitioning large tables, updating statistics, and using temp tables for intermediate results instead of CTEs.
4. A report has incorrect data after ETL. How do you investigate?
   - **Answer:** I would trace the data flow from source to report - check staging tables, transformation logic, and aggregation steps. In CDMS, an incorrect report was traced to a JOIN causing data duplication; adding DISTINCT and verifying row counts at each stage identified the issue.
   - **If asked more:** I would compare row counts at each ETL stage, check NULL handling, examine transformation SQL for logic errors, and compare against a manually calculated sample.
5. Duplicate serial records are entering inventory. How do you prevent it?
   - **Answer:** I would add a unique constraint on the serial number at the database level and a duplicate check in the application layer. In the inventory system, we prevented duplicates by checking the serial against the verified baseline and rejecting existing records.
   - **If asked more:** I would discuss handling race conditions with Serializable isolation level for check-then-insert, and logging rejected duplicates for auditing.
6. Inventory data is delayed from some channels. How do you handle it?
   - **Answer:** I would investigate the data flow for each delayed channel - check submission timestamps, ETL processing, and error logs. In our inventory system, delays were often from partners sending data in non-standard formats that failed validation.
   - **If asked more:** I would set up monitoring for each channel's submission time, create alerts for missed windows, and implement a grace period before marking data as stale.
7. Kafka consumer lag is increasing. How do you debug it?
   - **Answer:** I would check consumer lag metrics in Kafka monitoring, identify which partition has the highest lag, and check consumer logs for slow processing. In the cold-chain project, lag increased when InfluxDB writes bottlenecked during high sensor traffic.
   - **If asked more:** I would check consumer processing rate vs producer rate, look for poison pill messages, consider increasing partitions or consumer instances, and optimize processing with batch writes.
8. Kafka messages are duplicated. How do you handle it?
   - **Answer:** Duplicates are expected with at-least-once delivery. Our consumer was designed idempotently - processing the same message twice produced the same result. In the cold-chain project, we used deduplication IDs in InfluxDB to ignore duplicate sensor readings.
   - **If asked more:** I would discuss enabling idempotent producers, tracking processed message IDs in Redis, and designing business logic to tolerate duplicates.
9. Kafka messages are out of order. How do you handle it?
   - **Answer:** Out-of-order messages happen when producers retry or networks delay. Each sensor reading had a timestamp for sorting. For strict ordering, we used a partition key on sensor ID, ensuring messages for the same sensor stayed in order.
   - **If asked more:** I would discuss partition keys for per-entity ordering, handling late-arriving data with a tolerance window, and the tradeoff between ordering and parallelism.
10. A Kafka consumer keeps failing on one message. What do you do?
    - **Answer:** I would check consumer logs for the specific error and examine the problematic message. If corrupt, I would skip it using a dead-letter queue pattern. We configured a SeekToCurrentErrorHandler that retried a few times then sent to a DLQ topic.
    - **If asked more:** I would discuss implementing a dead-letter topic for failed messages, alerts for DLQ entries, and periodic review to fix upstream data issues.
11. AWS Lambda is timing out. How do you debug it?
    - **Answer:** I would check CloudWatch Logs, Lambda timeout configuration, and identify the slow operation. In the cold-chain project, a Lambda timed out because the downstream Kafka publish had network connectivity issues to the EC2 broker.
    - **If asked more:** I would increase timeout temporarily, check memory allocation (more memory = more CPU), optimize the code (connection reuse, reduce payload), and consider async invocation for long operations.
12. EC2 deployment failed after Jenkins pipeline ran successfully. What do you check?
    - **Answer:** I would SSH into EC2 and check Docker logs, disk space, and the application log. In our project, a deployment failed because the EC2 instance ran out of disk space from old Docker images not being cleaned up.
    - **If asked more:** I would check EC2 resource usage, verify the Docker daemon is running, check the compose file for correct image tags, and examine application startup logs for connection failures.
13. API response time increased suddenly. How do you debug it?
    - **Answer:** I would check recent deployments, database query performance, and external API dependencies. In the cold-chain project, response time increased when an InfluxDB query stopped using the time index after a frontend update changed the query filter, causing full scans.
    - **If asked more:** I would check APM tools or CloudWatch metrics, look for slow queries, thread pool exhaustion, Redis cache hit ratios, and identify slow endpoints via detailed request logging.
14. Database connections are exhausted. How do you debug it?
    - **Answer:** I would check the database's active connection list and identify which application or query is holding connections without releasing. In our Spring Boot app, a missing connection pool configuration for a long-running report query consumed all connections.
    - **If asked more:** I would configure HikariCP properly (max pool size, timeout, leak detection), review code for missing connection closes, and add monitoring on connection pool metrics via Actuator.
15. Redis cache has stale data. How do you fix it?
    - **Answer:** I would check TTL configuration and cache invalidation strategy. Stale data means TTL is too long or cache is not invalidated when source data changes. In our cold-chain project, we set appropriate TTLs and invalidated cache entries when new sensor data arrived.
    - **If asked more:** I would discuss publishing invalidation events on data updates, using shorter TTLs as a quick fix, and write-through caching for data that changes frequently.
16. A scheduled job runs on all pods and sends duplicate emails. How do you solve it?
    - **Answer:** I would use a distributed lock with Redis via Redisson or a database lock table, so only one instance acquires the lock and executes the job. We used `@SchedulerLock` where the first Spring Boot instance to acquire the lock ran the job.
    - **If asked more:** I would discuss lock TTL (lease time) to prevent deadlocks, graceful handling of lock acquisition failures, and monitoring which instance executed each run.
17. A scheduled job is missing executions. How do you monitor it?
    - **Answer:** I would add logging at job start and end with timestamps and create a check verifying the last successful execution time is within expected intervals. In our inventory system, a health check endpoint reported the last reconciliation job run time, alerting if overdue.
    - **If asked more:** I would discuss exposing job execution metrics via Actuator, creating a Grafana dashboard for success/failure rates, and configuring alerts for missed schedules.
18. JWT works locally but fails behind load balancer. What do you check?
    - **Answer:** I would check if the load balancer strips or modifies headers, particularly the Authorization header. In our cold-chain project, JWT failed on EC2 because SSL termination at the load balancer changed the protocol, and the `secure` flag on JWT cookies did not match.
    - **If asked more:** I would also check X-Forwarded-Proto and X-Forwarded-For headers and verify Spring Security's requiresSecure() matches the actual protocol.
19. Users get 403 after enabling CSRF. How do you fix it?
    - **Answer:** For our REST APIs used by a SPA, we disabled CSRF protection because it is not needed for token-based authentication - CSRF is primarily for cookie-based auth. We called `http.csrf().disable()` in Spring Security.
    - **If asked more:** I would explain that SPAs using JWT in Authorization headers are not vulnerable to CSRF since browsers do not auto-include Auth headers for cross-origin requests.
20. Admin role update is not reflected immediately. How do you design it?
    - **Answer:** I would ensure role changes either force re-login (if stored in JWT) or check from the database on every request. In our system, roles in JWT required re-login. For immediate effect, database checks on every API call would be needed.
    - **If asked more:** I would discuss the tradeoff: JWT roles are fast but not immediately updatable; database-checked roles are slower but real-time. A hybrid uses short-lived JWTs with DB fallback for sensitive operations.
21. Production API returns 500 but logs are unclear. How do you improve observability?
    - **Answer:** I would add structured logging with correlation IDs, ensure exception stack traces are logged, and log request/response bodies at TRACE level for debugging. In the cold-chain project, unclear logs were fixed by adding method-level logging with input parameters and context.
    - **If asked more:** I would discuss implementing Actuator for health checks, distributed tracing, centralized log aggregation (CloudWatch or ELK), and alerting based on error rate thresholds.
22. A service dependency is down. How should your service behave?
    - **Answer:** The service should degrade gracefully - return cached data if available, return a meaningful error message (not 500), and not crash or hang. In the cold-chain project, when InfluxDB was unavailable, the API returned stale data from Redis with a `stale: true` flag.
    - **If asked more:** I would discuss circuit breaker to fail fast, reasonable timeouts (2-3 seconds), and fallback responses that let the frontend show appropriate UI like a "data may be delayed" banner.
23. A dashboard becomes slow with live data. How do you optimize it?
    - **Answer:** I would optimize the backend (query optimization, pagination, caching) and the frontend (reduce re-renders, virtual lists, debounce updates). In our cold-chain dashboard, we updated data every 30 seconds instead of on every reading and pre-aggregated time-series data in InfluxDB.
    - **If asked more:** I would discuss WebSocket updates instead of polling, React.memo to prevent unnecessary re-renders, and server-side rendering for initial page load.
24. IoT sensor sends invalid readings. How do you validate them?
    - **Answer:** I would apply range checks (temperature between -40 and 100C), format checks (valid JSON, required fields), and rate-of-change checks (flag if temp jumps 20 degrees in one minute). In the cold-chain project, Lambda validated incoming MQTT messages before publishing to Kafka.
    - **If asked more:** I would discuss handling different invalid data types - sensor malfunction (constant values), communication errors (gaps, checksum failures), and out-of-range values - each with different strategy (reject, flag, or interpolate).
25. IoT gateway sends data in bursts. How do you handle backpressure?
    - **Answer:** Kafka acts as a buffer that absorbs bursts - producers publish at high rates while consumers process at their own pace. In the cold-chain project, when gateways reconnected and sent bursts, Kafka queued messages and consumers caught up gradually without data loss.
    - **If asked more:** I would discuss monitoring consumer lag to detect backpressure, scaling consumers by increasing partitions, and rate limiting at ingestion if bursts exceed Kafka capacity.
26. A database migration causes backlog. How do you recover safely?
    - **Answer:** I would stop writes to the affected table, assess backlog size, and run batch processing to catch up. For CDMS, if a migration caused failures, we would restore from backup with point-in-time recovery, fix the script, and re-run during low-traffic hours.
    - **If asked more:** I would discuss testing migrations on staging with production data, using backward-compatible migrations, and having a rollback script ready before running any migration.
27. A deployment needs rollback. What is your process?
    - **Answer:** I would stop the current deployment, restore the previous Docker image tag from Jenkins artifacts, and redeploy. We kept the last three successful image tags, so rollback was `docker pull app:v1.2.3-previous` and restarting containers, followed by smoke tests.
    - **If asked more:** I would differentiate code rollback vs database rollback - DB changes must be backward-compatible so code rollback is safe. For irreversible changes, deploy a fix forward instead.
28. A Docker container works locally but fails on EC2. What do you check?
    - **Answer:** I would compare environment variables, Docker image versions, and resource limits. In our project, a Spring Boot container failed on EC2 because the instance had less memory, causing the JVM to hit its limit and get OOM-killed by Docker.
    - **If asked more:** I would check EC2 docker logs, verify environment variables, check file permissions for mounted volumes, compare OS architecture, and ensure ports are open in the security group.
29. A REST API has inconsistent error responses. How do you standardize it?
    - **Answer:** I would implement global exception handling with `@ControllerAdvice` that catches all exceptions and returns a consistent JSON format: `{status, error, message, timestamp}`. In our projects, a custom ErrorResponse DTO ensured every error had the same structure.
    - **If asked more:** I would discuss defining error codes, field-level validation errors in standard format, and documenting error schemas in OpenAPI so the frontend handles errors generically.
30. A frontend user sees pages they should not access. How do you fix RBAC?
    - **Answer:** I would check both frontend route protection and backend authorization. The frontend might hide a button but not protect the route. In our dashboard, route guards checked user roles from JWT, and backend always enforced `@PreAuthorize` on APIs.
    - **If asked more:** I would discuss the principle: frontend RBAC is UX only, backend must always validate permissions. The fix adds server-side authorization checks matching frontend route guards.
31. A stored procedure returns correct results but is slow. What metrics do you check?
    - **Answer:** I would check the actual execution plan for index scans, key lookups, and sort operations, plus logical reads (high reads = inefficiency), wait stats (blocking, I/O), and estimated vs actual row counts.
    - **If asked more:** I would use SET STATISTICS TIME and IO for detailed metrics, compare with OPTION (RECOMPILE) to rule out parameter sniffing, and check if statistics need updating.
32. A query is fast for one parameter but slow for another. What could be wrong?
    - **Answer:** This is classic parameter sniffing - the optimizer creates a plan based on the first parameter value, which may be inefficient for others. In CDMS, a date-filtered procedure was fast for 2024 (small range) but slow for 2023 (large range).
    - **If asked more:** I would use OPTION (RECOMPILE) to test, check parameter data types match column types, look for skewed data distribution, and consider OPTION (OPTIMIZE FOR UNKNOWN).
33. A microservice is receiving duplicate requests. How do you make it idempotent?
    - **Answer:** I would check a unique request ID from the client before processing, store processed IDs in a database with a unique constraint, and return the existing result for duplicates. In our inventory system, the batch reference ID was the idempotency key.
    - **If asked more:** I would discuss idempotency key generation (UUID v4), storage (Redis with TTL or DB table), response caching, and TTL-based expiry for the key.
34. A payment-like operation succeeds but client times out. How do you handle retry?
    - **Answer:** I would design the operation to be idempotent with a unique transaction ID. On retry with the same ID, the server detects it is a duplicate and returns the saved result instead of processing again.
    - **If asked more:** I would discuss implementing a status check endpoint where the client polls for completion using the transaction ID, and webhooks for async notification.
35. A report must be generated exactly once across multiple servers. How do you design it?
    - **Answer:** I would use a distributed lock (Redis or database-based) that the first server acquires before generating. The lock prevents other servers from starting the same report. If the generating server crashes, the lock expires and another server retries.
    - **If asked more:** I would also use a database table tracking report status (submitted, in-progress, completed) with optimistic locking and scheduled cleanup for abandoned "in-progress" entries.
36. An API must process 10,000 records. Do you process synchronously or asynchronously?
    - **Answer:** Asynchronously. Synchronous processing would hold the HTTP connection for minutes, causing timeouts. I would return 202 Accepted with a job ID, process asynchronously via Kafka or a thread pool, and let the client poll a status endpoint.
    - **If asked more:** I would discuss the tradeoffs: async adds complexity but prevents timeouts and enables scaling. For smaller batches (<100 records), synchronous is simpler.
37. A customer asks for near real-time dashboard updates. What architecture would you choose?
    - **Answer:** I would use WebSocket or Server-Sent Events for push-based updates. In our cold-chain project, 30-second polling was sufficient, but for sub-second real-time, WebSocket is better - the backend pushes sensor updates to clients as they arrive from Kafka.
    - **If asked more:** I would discuss the pipeline: sensor -> Kafka -> Spring Boot (WebSocket broadcast) -> React. For scaling, Redis Pub/Sub coordinates broadcasts across multiple server instances.
38. A database table is growing very large. How do you manage performance?
    - **Answer:** I would implement table partitioning (by date for time-series), archive old data to cold storage, and ensure queries use partition elimination. InfluxDB handled this naturally with time-based shards; for MSSQL, I would partition by month with a data retention policy.
    - **If asked more:** I would discuss narrow indexes on queried columns, maintaining statistics, page compression for storage/I/O savings, and read replicas for reporting queries.
39. A third-party API is slow. How do you protect your application?
    - **Answer:** I would implement circuit breaker with timeout to fail fast, return cached data as fallback, and process third-party calls asynchronously with retry. In the cold-chain project, if the weather API dependency was slow, the dashboard returned last known weather with a "stale" indicator.
    - **If asked more:** I would discuss appropriate timeouts (connection + read), bulkhead isolation to prevent slow API calls from consuming all threads, and monitoring circuit breaker state.
40. A production issue happens at midnight. How do you approach debugging?
    - **Answer:** I would first check scheduled jobs and batch processes running at midnight - they often cause resource contention. In CDMS, midnight was when the ETL pipeline ran, so a midnight issue was likely ETL-related. I would check job logs, database waits, and system resources during that window.
    - **If asked more:** I would review recent deployments before midnight, check monitoring dashboards for the specific time, look for concurrent operations that might conflict, and reproduce by running jobs in staging with similar load.

---

## Questions To Ask Interviewer

1. What are the main responsibilities of this role?
   - **Answer:** I ask this to understand whether the role is backend-heavy or full-stack, and whether it aligns with my strength in Spring Boot and databases.
   - **If asked more:** I would use the answer to tailor follow-up questions about team structure and on-call expectations.
2. What kind of backend systems will I work on?
   - **Answer:** I ask this to judge if the systems match my experience - I want to work on data-intensive, high-throughput systems like the ones I have built.
   - **If asked more:** I would ask about scale, latency requirements, and if they face performance challenges similar to what I solved in CDMS.
3. What is the main technology stack used by the team?
   - **Answer:** I ask this to check if my Java, Spring Boot, SQL, Kafka, and AWS experience is directly applicable or if I need to learn new technologies.
   - **If asked more:** I would ask about the version of frameworks and if they are considering stack modernization.
4. Is the architecture monolithic, microservices-based, or hybrid?
   - **Answer:** I ask this to understand the complexity level and whether I can contribute to system design decisions beyond just writing code.
   - **If asked more:** If the answer is monolith, I would ask about any planned migration to microservices and their timeline.
5. What are the biggest technical challenges the team is solving now?
   - **Answer:** I ask this to see if the challenges match my strengths - performance optimization, data pipelines, or real-time processing.
   - **If asked more:** I would share a similar challenge from my experience (like CDMS optimization) to show I can contribute immediately.
6. What are the expectations in the first 3 months?
   - **Answer:** I ask this to understand the onboarding process and what success looks like early on, so I can prioritize my learning effectively.
   - **If asked more:** I would ask about codebase size, documentation quality, and who would be my primary mentor during onboarding.
7. How does the team handle code reviews?
   - **Answer:** I ask this because a good code review culture is important for my growth and code quality. I want to know if reviews are thorough or just rubber-stamping.
   - **If asked more:** I would ask about typical review turnaround time and whether performance considerations are discussed during reviews.
8. How does the team handle deployments?
   - **Answer:** I ask this to understand deployment frequency, automation level, and whether they use CI/CD similar to my Jenkins setup.
   - **If asked more:** I would ask about their deployment strategy (rolling, blue-green, canary) and how they handle failed deployments.
9. What CI/CD tools are used?
   - **Answer:** I ask this to see if they use Jenkins, GitHub Actions, or other tools, and whether my CI/CD experience transfers directly.
   - **If asked more:** I would ask about pipeline stages, how tests are integrated, and whether they use infrastructure as code.
10. How are production issues monitored?
    - **Answer:** I ask this because good observability is critical for debugging - I want to know if they have dashboards, alerts, and centralized logging.
    - **If asked more:** I would ask about their on-call rotation, incident response process, and postmortem culture.
11. What observability tools are used?
    - **Answer:** I ask this to check if they use tools I have experience with (CloudWatch, Grafana) or if I would need to learn new monitoring stacks.
    - **If asked more:** I would ask about logging aggregation, metric collection, and distributed tracing capabilities.
12. How is ownership divided across services?
    - **Answer:** I ask this to understand team structure and whether I would own specific services end-to-end or work across multiple areas.
    - **If asked more:** I would ask about team size per service and how cross-service coordination works.
13. Does the team follow Agile/Scrum?
    - **Answer:** I ask this to understand the work cadence and whether I will need to adapt to sprint planning, stand-ups, and retrospectives.
    - **If asked more:** I would ask about sprint length, planning process, and how technical tasks are prioritized vs feature work.
14. How much backend vs frontend work is expected?
    - **Answer:** I ask this to ensure the role is primarily backend, which is my strength and preference, rather than full-stack.
    - **If asked more:** I would clarify that I can do frontend when needed but want a backend-focused growth path.
15. Are there opportunities to work on system design and architecture?
    - **Answer:** I ask this because I want to grow beyond implementation into design decisions, which aligns with my 3-5 year career goal.
    - **If asked more:** I would ask about the team's approach to technical decision-making and whether junior engineers participate in design discussions.
16. How does the team handle technical debt?
    - **Answer:** I ask this to understand whether the team prioritizes code quality and refactoring or focuses only on feature delivery.
    - **If asked more:** I would ask about their process for identifying and prioritizing technical debt and if there is dedicated time for refactoring.
17. How are performance issues handled?
    - **Answer:** I ask this because performance optimization is one of my strengths from CDMS, and I want to work where this skill is valued.
    - **If asked more:** I would ask about their performance testing process, tools used, and whether performance is a consideration in code reviews.
18. What databases and messaging systems are used?
    - **Answer:** I ask this to check if they use MSSQL/PostgreSQL and Kafka, which match my experience, or different technologies I would need to learn.
    - **If asked more:** I would ask about data volume, query patterns, and whether they have experienced performance issues similar to what I solved.
19. What cloud platform is used?
    - **Answer:** I ask this to see if they use AWS (my experience) or another cloud provider, and whether my EC2, ALB, and IoT Core skills are relevant.
    - **If asked more:** I would ask about their cloud migration journey, multi-region strategy, and how they manage cloud costs.
20. What does success look like for this role?
    - **Answer:** I ask this to understand how my performance will be evaluated beyond just coding - whether it is about delivery speed, quality, ownership, or business impact.
    - **If asked more:** I would ask about promotion criteria and what the career progression path looks like for backend engineers.

