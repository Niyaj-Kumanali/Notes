# Personalized Questions

---

**Q. Can you describe how you optimized database performance and query execution in your work at Talentpace Pvt Ltd?**

In my role at Talentpace Pvt Ltd, one of the major areas I worked on was Microsoft SQL Server performance optimization.

One important project was the Channel Data Management System for Lenovo India. In that system, some of the legacy stored procedures were taking almost 5 hours to complete, especially for daily reporting and large channel data processing.

My first step was to analyze the execution plans and understand where the bottlenecks were. I checked for missing indexes, expensive joins, table scans, and places where the stored procedures were doing unnecessary processing. Based on that, I refactored more than 20 stored procedures, rewrote some of the queries, and improved the indexing strategy.

After these optimizations, the execution time came down from around 5 hours to under 12 minutes. This made a big difference because the reporting became much faster and more reliable for the business team.

I also worked on the Real-Time Partner Inventory Management System, where I built a centralized validation engine for inventory calculations. In that project also, I used optimized MSSQL queries to reduce batch processing time to under 15 seconds and improve inventory accuracy to around 94% across 200+ channels.

So overall, my approach was not just to write queries that work, but to check how they behave with real data volume, analyze execution plans, tune indexes, and test them under practical load.

---

**Q. How have you built scalable data processing pipelines using technologies like AWS and Apache Kafka?**

At Talentpace Pvt Ltd, I worked on a Cold-Chain Monitoring System where we had to process live IoT data coming from more than 100 gateways.

The system used AWS IoT Core for receiving device data, AWS Lambda for processing events, Apache Kafka for streaming, InfluxDB for storing time-series data, and Grafana for dashboards and monitoring.

Kafka was useful because the data was continuously coming from many gateways, and we did not want the whole system to depend on direct synchronous processing. By using Kafka, we were able to decouple data ingestion from downstream processing. If the data volume increased, consumers could be scaled independently.

I also worked with Kafka partitioning and replication so that the load could be distributed properly and the pipeline would remain reliable even when traffic increased. The processed metrics were stored in InfluxDB, which was a good fit for sensor and time-series data, and Grafana helped us monitor the system health in real time.

This architecture helped reduce data load time by around 80% and improved overall responsiveness by about 30%.

My role involved backend development, pipeline integration, query optimization, and working with the team to make sure the data flow was reliable from device ingestion to dashboard visualization.

---

**Q. Tell me about a backend improvement you made that had measurable business impact.**

One example I can talk about is the Real-Time Partner Inventory Management System.

In this project, inventory data was coming from more than 200 channels, and the validation logic was very important because incorrect inventory data could directly affect reporting and decision-making.

I worked on building a centralized validation engine that automated inventory calculations and made the validation process more consistent. I used optimized MSSQL queries and standardized the calculation rules so that the system could process inventory batches faster and with better accuracy.

After this improvement, batch processing time came down to under 15 seconds, and inventory accuracy improved to around 94%.

The main business impact was that the operations team had faster and more reliable inventory visibility, and there was less dependency on manual checking or correction.

---

**Q. What is one of your strongest technical contributions?**

One of my strongest technical contributions was optimizing the Lenovo India Channel Data Management System.

The challenge was that some important stored procedures were taking close to 5 hours, which affected daily reporting. I analyzed the execution plans, found inefficient joins and missing indexes, and refactored more than 20 stored procedures.

After the optimization, the same process completed in under 12 minutes. I consider this one of my strongest contributions because it was not just a code change. It had a clear business impact, improved reporting speed, and made the system more dependable for daily operations.

---

**Q. How do you usually approach performance issues in backend systems?**

My approach is to first measure and understand the actual bottleneck instead of directly changing the code.

For example, if it is a database performance issue, I check the execution plan, query cost, index usage, joins, scans, and data volume. If it is an API issue, I check logs, response time, database calls, external service calls, and whether there is any unnecessary processing.

After identifying the bottleneck, I make focused changes like query rewriting, indexing, caching, batching, or reducing unnecessary calls. Then I compare before and after results to make sure the change actually improved performance and did not affect correctness.

This is the same approach I followed in my SQL Server optimization work, where we reduced execution time from around 5 hours to under 12 minutes.

---

**Q. Give me a short introduction about your backend experience.**

I have backend development experience mainly around Java, Spring Boot, Microsoft SQL Server, Kafka, and AWS-based systems.

At Talentpace Pvt Ltd, I worked on projects involving database optimization, real-time inventory validation, and IoT data pipelines. One of my key contributions was optimizing legacy stored procedures for the Lenovo India Channel Data Management System, where I reduced execution time from around 5 hours to under 12 minutes.

I have also worked on real-time data processing using AWS IoT Core, Lambda, Kafka, InfluxDB, and Grafana for a Cold-Chain Monitoring System.

My strength is backend development where performance, reliability, and data correctness are important.

---

## HR Questions

**Q. Tell me about yourself.**

My name is Niyaj Kumanali. I am a Software Engineer with around 2.5+ years of experience, mainly working on backend development, database optimization, and real-time data processing systems.

Currently, I am working at Talentpace Pvt Ltd in Bengaluru. My main experience is with Java, Spring Boot, Microsoft SQL Server, Kafka, AWS, and also some frontend work using React.

In my current role, I have worked on projects for clients like Lenovo India and UrjaLinks. One of my key contributions was optimizing legacy stored procedures in the Lenovo Channel Data Management System, where we reduced execution time from around 5 hours to under 12 minutes. I also worked on real-time inventory validation across 200+ channel partners and an IoT-based cold-chain monitoring system using AWS IoT Core, Lambda, Kafka, InfluxDB, and Grafana.

Overall, I would describe myself as someone who is strong in backend development, performance tuning, and building systems where data correctness and reliability are important.

---

**Q. Why are you looking for a change?**

I am looking for a change because I want to grow into a role where I can work on larger-scale backend systems and take more ownership in design and development.

In my current role, I have got good exposure to database optimization, real-time pipelines, Spring Boot, Kafka, and AWS. Now I want to build on that experience and work in an environment where I can contribute to more complex backend architecture, microservices, and scalable product development.

My reason is mainly career growth, better technical exposure, and the opportunity to work on challenging engineering problems.

---

**Q. Why should we hire you?**

You should hire me because I bring practical backend experience, not just theoretical knowledge.

I have worked on real production problems where performance, data accuracy, and reliability mattered. For example, I optimized MSSQL stored procedures and reduced execution time from around 5 hours to under 12 minutes. I also worked on inventory validation systems processing data across 200+ channels, and real-time IoT pipelines using AWS and Kafka.

Along with Java and Spring Boot, I have hands-on experience with databases, query optimization, Kafka, Redis, AWS, Jenkins, Docker, and React. So I can contribute across different parts of the application, but my strongest area is backend engineering.

I am also comfortable debugging issues, understanding existing systems, and improving them step by step.

---

**Q. What are your strengths?**

My main strength is problem-solving in backend systems.

I usually try to understand the root cause before making changes. For example, in database performance issues, I do not directly add indexes or rewrite queries. I first check execution plans, data volume, joins, scans, and actual bottlenecks. That helped me reduce one reporting process from around 5 hours to under 12 minutes.

Another strength is that I can work across backend, database, and integration areas. I have worked with Spring Boot APIs, MSSQL optimization, Kafka pipelines, AWS services, and React-based dashboards.

I also take ownership of my work and try to make sure the solution is reliable, not just completed.

---

**Q. What is your weakness?**

One area I have been improving is that earlier I used to spend extra time trying to make a solution perfect before sharing it.

Over time, I understood that in real projects, it is better to first make the solution clear, testable, and useful, then improve it based on feedback. Now I try to break work into smaller parts, discuss early if there is any confusion, and take feedback before going too deep in one direction.

This has helped me work better with the team and deliver faster without compromising quality.

---

**Q. Where do you see yourself in the next 3 to 5 years?**

In the next 3 to 5 years, I want to grow into a strong backend engineer who can design and own scalable systems end to end.

Right now, I have good experience in Java, Spring Boot, databases, Kafka, and AWS-based systems. I want to continue improving in system design, microservices architecture, cloud deployment, and performance engineering.

My goal is to become someone who can not only implement features, but also make good design decisions, mentor juniors, and take ownership of critical backend services.

---

**Q. Why do you want to join our company?**

I want to join because I am looking for a role where I can work on strong engineering problems and grow technically.

Based on the role, I feel my experience in Java, Spring Boot, database optimization, Kafka, AWS, and real-time systems is relevant. I have already worked on production systems where performance and reliability were important, and I want to contribute that experience in a larger or more product-focused environment.

For me, the main things I look for are good technical work, learning opportunities, ownership, and a team where I can contribute meaningfully.

---

**Q. Are you comfortable working in a team?**

Yes, I am comfortable working in a team.

In my current role, most of my work involved coordination with backend developers, frontend developers, QA, and sometimes business or operations teams. For example, in the cold-chain monitoring system, the backend pipeline, dashboard, alerts, and deployment all needed coordination across different areas.

I try to communicate clearly, give updates on blockers, and understand the impact of my work on other team members. I am also comfortable taking feedback during code reviews and improving the implementation.

---

**Q. Tell me about a challenging situation you handled.**

One challenging situation was in the Lenovo Channel Data Management System, where some stored procedures were taking close to 5 hours.

The challenge was that these procedures were part of an existing system, so I had to improve performance without affecting the correctness of daily reports. I analyzed execution plans, identified bottlenecks like inefficient joins and missing indexes, and refactored more than 20 stored procedures.

After optimization, the execution time reduced to under 12 minutes. The challenging part was not only improving performance, but also validating that the reporting output was still correct after the changes.

---

**Q. Tell me about a time you made a mistake.**

In the initial stage of my work, one mistake I made was focusing more on the implementation first and not always documenting the reasoning behind some technical changes clearly enough.

Later, I realized that in team projects, especially when working on database procedures or backend logic, others should be able to understand why a change was made. So I improved the way I communicate changes by adding clearer comments where needed, sharing the before-and-after impact, and explaining the logic during reviews.

That helped the team review my work faster and also made future maintenance easier.

---

**Q. How do you handle pressure or tight deadlines?**

When there is pressure or a tight deadline, I first try to break the work into smaller parts and identify what is most critical.

I focus on completing the high-priority work first, and I keep the team updated if there is any blocker or risk. I also avoid making random changes under pressure. Even if the timeline is tight, I try to test the important flows properly, especially when the work is related to database logic or production APIs.

In my experience, clear prioritization and communication help reduce pressure and avoid last-minute surprises.

---

**Q. Are you more interested in backend or full-stack development?**

My strongest interest is backend development.

I enjoy working on APIs, database optimization, system design, Kafka pipelines, and performance-related problems. At the same time, I also have experience with React and frontend dashboards, so I can contribute in full-stack work when needed.

But if I have to choose my core strength, I would say backend engineering, especially Java, Spring Boot, databases, and scalable systems.

---

**Q. What motivates you at work?**

I am motivated when I can see that my work has a clear impact.

For example, reducing a process from 5 hours to under 12 minutes, improving inventory accuracy, or making real-time dashboards more responsive gives me confidence that my work is useful for the business and the users.

I also enjoy learning new technical areas and applying them practically, especially in backend performance, distributed systems, and cloud-based applications.

---

**Q. How do you handle feedback?**

I take feedback positively because it helps me improve.

In development work, feedback is important because sometimes another developer may notice a better approach, edge case, or maintainability issue. I try to understand the reason behind the feedback and apply it properly instead of just making the requested change.

Code reviews have helped me improve my coding style, design thinking, and testing approach.

---

**Q. What kind of work environment do you prefer?**

I prefer an environment where there is clear communication, ownership, and good technical learning.

I like working with teams where people discuss solutions, review code properly, and focus on building reliable systems. I am comfortable with Agile/Scrum process, regular updates, and working with cross-functional teams.

For me, a good environment is one where I can contribute, learn, and take responsibility for meaningful work.

---

**Q. What is your expected salary?**

I am open to discussing compensation based on the role, responsibilities, and company standards.

My main focus is on finding the right opportunity where I can contribute well and grow technically. Based on my experience and the market range for this role, I am expecting a fair compensation, but I am flexible for the right opportunity.

---

**Q. What is your notice period?**

My notice period is as per my current company policy. I can discuss the exact timeline based on the offer process and joining requirement.

If needed, I can also check internally for possibilities of early release after the offer stage.

---

**Q. Do you have any questions for us?**

Yes, I would like to understand more about the role and the team.

What kind of backend systems or products will I be working on?

What are the main technologies used by the team?

What are the expectations from this role in the first 3 to 6 months?

Also, I would like to know how the team handles code reviews, deployments, and technical ownership.
