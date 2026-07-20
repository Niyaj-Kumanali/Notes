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
