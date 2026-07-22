# C# Interview Questions With Answers

This file is based on `CSharp-Interview-Questions-Bank.md`. Answers are written in a longer, spoken first-person style based on Abdul Shoaib's resume.

Note: The complete generator for all 615 questions is available at `.build-csharp-answers.js`, but the command runner failed before it could execute in this turn. I started the answered file with the most interview-critical personalized sections.

---

## HR Questions

1. Tell me about yourself.
   - **Answer:** My name is Abdul Shoaib. I am a Software Engineer with around 2.8+ years of experience, mainly in C#, ASP.NET Core Web API, SQL Server, Entity Framework Core, Dapper, Redis, Azure DevOps, IIS, and full-stack development. In my current role at Talentpace Pvt Ltd, I have worked on enterprise applications for Lenovo Malaysia and procurement automation. My work includes building APIs, optimizing SQL Server stored procedures, implementing RBAC, integrating LDAP-based SSO, improving API latency, automating approval workflows, and deploying applications through IIS and CI/CD pipelines. I would describe myself as a full-stack developer with strong backend focus, especially in API performance, workflow automation, and database optimization.

2. Walk me through your resume.
   - **Answer:** I started my professional career as a Software Engineer at Talentpace Pvt Ltd. My resume mainly highlights enterprise application development using C#, ASP.NET Core Web API, SQL Server, Angular, React, Redis, Azure DevOps, Docker, and IIS. In the Demo Request Management System for Lenovo Malaysia, I delivered an Angular and ASP.NET Core full-stack platform, integrated LDAP-based SSO, implemented RBAC, managed IIS hosting, and optimized API latency from 100 seconds to 30 seconds. In the End User Verification project, I designed validation APIs and tuned stored procedures for compliance checks. In PROXPERT, I worked on modular ASP.NET Core backend services, Redis caching, CQRS, OAuth2/JWT security, dynamic RBAC, CMS, Docker, and Azure DevOps CI/CD. Overall, my resume shows full-stack delivery, backend performance tuning, workflow automation, and production ownership.

3. Why are you looking for a change?
   - **Answer:** I am looking for a change mainly for career growth and stronger technical exposure. In my current role, I got good hands-on experience in enterprise applications, API design, SQL Server optimization, RBAC, workflow automation, Redis caching, IIS hosting, and Azure DevOps pipelines. Now I want to work on larger and more scalable systems, improve more in architecture, cloud, distributed systems, and take more ownership in backend and full-stack product development. My reason is not negative; it is mainly about growth, learning, and working on more challenging engineering problems.

4. Why should we hire you?
   - **Answer:** You should hire me because I have practical experience in building and improving production-grade enterprise applications. I have not only developed APIs and UI screens, but also improved measurable business outcomes. For example, I reduced Loan Listing API latency by 70%, automated approval workflows, enforced RBAC across 2,100+ users, improved concurrent request capacity by 57%, and implemented Redis caching to remove disk I/O bottlenecks. I can contribute in backend, database, deployment, and frontend areas, but my strongest value is solving real performance and workflow problems in production systems.

5. What are your strengths as a software engineer?
   - **Answer:** My main strengths are ownership, debugging, and performance-focused development. I try to understand the root cause before making changes. For example, when an API is slow, I do not only check code; I also check stored procedures, indexes, data volume, logs, and hosting configuration. Another strength is that I can handle full-stack work. In DRMS, I delivered Angular frontend, ASP.NET Core backend, LDAP SSO integration, RBAC, and IIS hosting, so I understand the complete application flow.

6. What is one weakness you are working on?
   - **Answer:** One weakness I have worked on is that earlier I used to spend more time trying to solve everything independently before discussing it. Later I realized that in enterprise projects, early communication is important, especially when requirements involve approvals, RBAC, or client-specific workflows. Now I still try to solve problems myself, but I also communicate early if there is a blocker or if a design decision needs confirmation. This has helped me deliver faster and avoid rework.

7. What motivates you at work?
   - **Answer:** I am motivated when I can see that my work has a real impact. For example, reducing API latency from 100 seconds to 30 seconds, automating manual approval workflows, improving concurrent request capacity, or reducing procurement timelines gives me confidence that my work is useful for the business. I also enjoy working on systems where performance, security, and workflow correctness matter, because those problems require both technical understanding and business context.

8. Where do you see yourself in the next 3 to 5 years?
   - **Answer:** In the next 3 to 5 years, I want to grow into a strong senior full-stack or backend engineer who can own critical services end to end. I want to improve more in system design, scalable architecture, cloud-native development, performance engineering, and technical leadership. My goal is to become someone who can not only implement features, but also make good design decisions, mentor juniors, and take ownership of production systems.

9. What kind of role are you looking for?
   - **Answer:** I am looking for a role where I can work on C#, ASP.NET Core, SQL Server, cloud, APIs, and full-stack enterprise systems. I am especially interested in backend-heavy roles where performance, database design, authentication, RBAC, workflow automation, and production reliability are important. I can contribute on the frontend also, because I have worked with Angular and React, but my stronger interest is backend and full-stack engineering with clear business impact.

10. Do you prefer backend, frontend, or full-stack development?
   - **Answer:** I prefer full-stack development with a strong backend focus. I like backend work because it involves APIs, databases, performance, authentication, caching, and system design. At the same time, I have delivered Angular and React-based frontend work, so I understand how frontend consumes APIs and how user workflows are built. This helps me design APIs better because I can think from both backend and frontend perspectives.

---

## Resume and Project Questions

1. Explain your current role at Talentpace Pvt Ltd.
   - **Answer:** Currently, I work as a Software Engineer at Talentpace Pvt Ltd in Bengaluru. My role includes building ASP.NET Core Web APIs, working on Angular and React frontends, optimizing SQL Server stored procedures, implementing RBAC and authentication, writing validation workflows, handling Redis caching, and supporting deployments through IIS and Azure DevOps. I have worked on enterprise applications for Lenovo Malaysia and procurement automation, where the focus was not only feature development but also performance, security, workflow correctness, and business impact.

2. What are your main responsibilities as a Software Engineer?
   - **Answer:** My responsibilities include understanding requirements, designing APIs, implementing backend logic, writing SQL queries and stored procedures, optimizing slow APIs, integrating authentication and authorization, building frontend screens, writing unit tests, managing logs, and supporting deployments. In some projects, I also handled full-stack delivery independently, including Angular frontend, ASP.NET Core backend, LDAP SSO integration, RBAC, and IIS production hosting. So my role is a mix of development, debugging, optimization, deployment, and coordination with teams.

3. Which project from your resume are you most confident about?
   - **Answer:** I am most confident about the Demo Request Management System because I worked on it end to end. I delivered the Angular and ASP.NET Core full-stack platform, integrated LDAP-based SSO, implemented RBAC, optimized stored procedures, managed IIS hosting, and worked on approval workflow features like mid-approval cancellation and Outlook Actionable Messages. It is a strong project for me because I can explain the business flow, architecture, database optimization, authentication, authorization, logging, and deployment parts clearly.

4. Which project had the highest business impact?
   - **Answer:** DRMS had a strong business impact because it directly improved approval workflow performance for Lenovo Malaysia users. The Loan Listing API latency was reduced from around 100 seconds to 30 seconds, which helped 600+ active approvers. The system supported 1,200+ requestors and enforced RBAC across 2,100+ users. PROXPERT also had high impact because it compressed procurement timelines by 86% and reduced processing time by 67% across 120 enterprise projects. If I had to choose one, I would highlight DRMS for direct user impact and PROXPERT for process automation impact.

5. Which project was technically the most challenging?
   - **Answer:** DRMS was technically challenging because I handled it end to end, from Angular frontend to ASP.NET Core APIs, SQL Server stored procedures, LDAP SSO, RBAC, IIS hosting, logging, and approval workflow logic. The challenge was not just building features; it was making sure the workflow behaved correctly for different roles and approval stages. Mid-approval cancellation and email-based approvals also required careful handling because duplicate actions, unauthorized access, and audit history had to be considered.

6. Explain the Demo Request Management System.
   - **Answer:** The Demo Request Management System, or DRMS, was built for Lenovo Malaysia to manage demo request and approval workflows. It supported requestors and approvers, with multi-stage approvals and role-based access control. I worked on the Angular frontend and ASP.NET Core Web API backend, integrated LDAP-based SSO, optimized SQL Server stored procedures, implemented mid-approval cancellation, added Outlook Actionable Messages for email-based approvals, and managed IIS hosting. The key impact was reducing Loan Listing API latency by 70%, reducing system exceptions, and enforcing RBAC across 2,100+ users.

7. What problem did DRMS solve for Lenovo Malaysia?
   - **Answer:** DRMS solved the problem of managing demo request approvals in a structured and secure way. Before automation, approval workflows could become slow, manual, and difficult to track. DRMS provided a platform where requestors could raise requests, approvers could take actions, and access could be controlled based on roles and approval stages. It also improved performance by reducing Loan Listing API latency and improved workflow efficiency through email-based approvals and mid-approval cancellation.

8. What was your role in DRMS?
   - **Answer:** My role in DRMS was end-to-end full-stack development. I worked on Angular frontend screens, ASP.NET Core Web APIs, SQL Server stored procedure optimization, LDAP-based SSO integration, RBAC implementation, Outlook Actionable Messages, mid-approval cancellation, logging with Serilog, unit testing with xUnit, and production hosting on IIS. Since I delivered the platform solo, I had to understand the complete flow from user action in the UI to backend validation, database update, approval state changes, and production deployment.

9. What does "end-to-end Angular + ASP.NET Core full-stack platform solo" mean in your resume?
   - **Answer:** It means I was responsible for delivering both frontend and backend parts of the DRMS platform independently. On the frontend side, I worked with Angular to build screens, forms, approval actions, and role-based UI behavior. On the backend side, I built ASP.NET Core Web APIs, handled business logic, integrated LDAP authentication, implemented RBAC, connected to SQL Server, optimized stored procedures, and managed IIS hosting. End-to-end also means I had to make sure the complete user flow worked properly, not only one layer.

10. How did you reduce Loan Listing API latency from 100 seconds to 30 seconds?
   - **Answer:** I started by identifying where the time was being spent. Since the Loan Listing API depended heavily on SQL Server stored procedures, I analyzed the database side first. I reviewed stored procedure logic, checked execution plans, looked for table scans, expensive joins, missing indexes, and unnecessary data processing. Then I refactored 11 legacy stored procedures and optimized the query flow. After the changes, I validated that the API still returned correct business data and measured the performance improvement. As a result, latency reduced from around 100 seconds to 30 seconds.

---

## Answering Pattern For Remaining Questions

The full bank has 615 questions. For the remaining technical sections, use this answer style while practicing:

1. Start with the simple definition.
2. Explain why it matters in real projects.
3. Mention where it fits in Abdul's resume: ASP.NET Core, SQL Server, EF Core, Dapper, Redis, Azure DevOps, IIS, RBAC, or workflow automation.
4. Add one practical example.
5. Add one common mistake or production consideration.

Example:

**Q. What is Redis?**

Redis is an in-memory data store used mainly for caching, fast lookup, distributed state, and sometimes pub/sub or queues. In my project PROXPERT, Redis was useful for concurrent security token processing for 500+ daily users. Earlier, token-related processing was creating disk I/O bottlenecks, so Redis helped by keeping frequently accessed temporary data in memory with TTL management. If I explain more, I would also mention cache expiry, stale data risk, what data should not be cached, and how we should secure cached token-related data.
