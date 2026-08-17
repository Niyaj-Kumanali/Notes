# AWS and Cloud Questions

## Questions

1. What AWS services have you used?
2. What is EC2?
3. What is S3?
4. What is Lambda?
5. What is AWS IoT Core?
6. Why use AWS IoT Core?
7. How does MQTT work with AWS IoT Core?
8. Difference between EC2 and Lambda.
9. When would you use Lambda?
10. When would you avoid Lambda?
11. What is serverless?
12. What is IAM?
13. What is IAM role?
14. What is IAM policy?
15. What is VPC?
16. What is subnet?
17. What is security group?
18. What is load balancer?
19. What is auto scaling?
20. What is CloudWatch?
21. How do you monitor AWS applications?
22. How do you store files in S3?
23. What is S3 bucket policy?
24. How do you secure S3?
25. How do you deploy Spring Boot on EC2?
26. How do you run Kafka on EC2?
27. What are the challenges of running Kafka on EC2?
28. How do you manage environment variables in AWS?
29. How do you handle secrets?
30. What is CI/CD deployment to AWS?
31. How did Jenkins deploy to EC2 in your project?
32. How do you debug AWS deployment failure?
33. What is high availability?
34. What is fault tolerance?
35. What is horizontal scaling?
36. What is vertical scaling?
37. What is Databricks?
38. How did Databricks help in predictive analytics?
39. What is data lake?
40. What is cloud cost optimization?

---

## Answers

1. What AWS services have you used?
   - **Answer:**
      - I've used EC2 (Spring Boot + Kafka deployment), IoT Core (MQTT gateway ingestion), Lambda (lightweight data transformation), S3 (report storage), IAM (access control), and CloudWatch (monitoring and logging)
      - Each was chosen for a specific role in the cold-chain and CDMS projects
   - **If asked more:**
      - I would go service by service — why I chose EC2 over Lambda for Kafka, how IoT Core handled device authentication, and how CloudWatch alarms alerted us to temperature excursions
2. What is EC2?
   - **Answer:**
      - EC2 is AWS's virtual server service
      - We deployed our Spring Boot APIs and Kafka brokers on EC2 instances, configured security groups for network access, and used auto-scaling to handle load
      - For Kafka, we chose EC2 over MSK to have full control over broker configuration
   - **If asked more:**
      - I would explain the instance sizing decisions — we used m5.large for Spring Boot and m5.xlarge for Kafka brokers based on memory and I/O requirements
3. What is S3?
   - **Answer:**
      - S3 is AWS's object storage service for storing and retrieving any amount of data
      - In CDMS, I used S3 to store generated reports — the optimized stored procedure output was written to S3 as CSV files that partners could download through the portal
   - **If asked more:**
      - I would discuss S3 storage classes — we used Standard for recent reports and transitioned older ones to Glacier for cost savings
4. What is Lambda?
   - **Answer:**
      - Lambda is AWS's serverless compute service that runs code on demand
      - In the cold-chain pipeline, I used a Lambda function to transform MQTT messages from AWS IoT Core and publish them to Kafka
      - The Lambda was triggered by an IoT Core rule and ran for under a second per invocation
   - **If asked more:**
      - I would explain how I configured the Lambda's memory (256MB) and timeout (30 seconds) based on the average message size, and how I handled cold start by keeping a warm pool
5. What is AWS IoT Core?
   - **Answer:**
      - AWS IoT Core is a managed cloud service that lets IoT devices connect and interact with AWS applications via MQTT
      - In cold-chain, IoT Core authenticated each gateway using device certificates, received temperature readings via MQTT, and routed them to Lambda through a rule
   - **If asked more:**
      - I would explain how IoT Core's device shadows stored the last known state of each gateway, which helped detect when a gateway went offline unexpectedly
6. Why use AWS IoT Core?
   - **Answer:**
      - We used IoT Core because it handles the heavy lifting of MQTT broker management, device authentication via X.509 certificates, and scales automatically with the number of connected gateways
      - Setting up a custom MQTT broker on EC2 would have required more operational effort
   - **If asked more:**
      - I would compare IoT Core with running a self-managed Mosquitto broker — IoT Core's device registry and policy engine saved us from building device management from scratch
7. How does MQTT work with AWS IoT Core?
   - **Answer:**
      - IoT gateways establish persistent MQTT connections to AWS IoT Core using device certificates
      - The gateways publish temperature readings to a topic like `sensors/gateway1/temperature`, and IoT Core triggers a rule that forwards the message to Lambda for further processing
   - **If asked more:**
      - I would explain MQTT QoS levels — we used QoS 1 (at-least-once) for reliable delivery — and how IoT Core's topic filters allowed us to subscribe to all gateways with a wildcard pattern
8. Difference between EC2 and Lambda.
   - **Answer:**
      - EC2 provides full control over the OS and runtime, suitable for stateful or long-running services like Kafka
      - Lambda is ephemeral and event-driven, ideal for short stateless tasks
      - I chose EC2 for Kafka because it needs persistent storage and continuous operation, and Lambda for message transformation because it runs only when data arrives
   - **If asked more:**
      - I would explain how Lambda's 15-minute timeout and 10GB storage limit made it unsuitable for Kafka, which needs sustained throughput and disk I/O
9. When would you use Lambda?
   - **Answer:**
      - Use Lambda for event-driven, short-lived tasks like transforming data between services, resizing images, or responding to API Gateway requests
      - In cold-chain, Lambda was perfect for converting MQTT messages to Kafka-compatible JSON — a simple, stateless transformation that ran for milliseconds per event
   - **If asked more:**
      - I would explain when Lambda is not a good fit: long-running processes (>15 min), stateful processing, or high-throughput sustained workloads where cost becomes unpredictable
10. When would you avoid Lambda?
    - **Answer:**
       - Avoid Lambda for stateful services, long-running computations, or workloads requiring consistent low latency
       - In cold-chain, we avoided Lambda for the main data processing pipeline because we needed sustained processing of 1000+ msg/sec and Kafka consumers are better suited for that
    - **If asked more:**
       - I would also mention cost — Lambda can be more expensive than EC2 for high-utilization workloads, and cold starts add latency that is unacceptable for real-time dashboards
11. What is serverless?
    - **Answer:**
       - Serverless means you don't manage servers — AWS handles scaling, patching, and availability
       - Lambda, IoT Core, and S3 are serverless services we used
       - Serverless is great for variable workloads because you pay only for what you use, but less suitable for predictable, high-utilization services like Kafka
    - **If asked more:**
       - I would explain the trade-off: serverless reduces operational overhead but can lead to unpredictable costs and cold-start latency, so we used a hybrid approach (serverless for ingestion, EC2 for processing)
12. What is IAM?
    - **Answer:**
       - IAM (Identity and Access Management) controls who can access AWS resources and what they can do
       - In cold-chain, I created IAM roles with least-privilege policies — Lambda had permission only to publish to the Kafka topic, and EC2 instances had access only to CloudWatch logs
    - **If asked more:**
       - I would explain the difference between IAM users (for people) and IAM roles (for services), and how I debugged access denied errors using CloudTrail
13. What is IAM role?
    - **Answer:**
       - An IAM role is an identity that AWS services assume to get temporary permissions
       - In cold-chain, the Lambda function had an IAM role with policies that allowed it to receive messages from IoT Core and publish to the Kafka topic
       - This avoided hardcoding any credentials
    - **If asked more:**
       - I would explain how I configured the trust policy to allow Lambda to assume the role, and how temporary credentials are rotated automatically by AWS
14. What is IAM policy?
    - **Answer:**
       - An IAM policy defines specific permissions in JSON format
       - I wrote a policy that allowed the Lambda to `kafka-cluster:Connect` and `kafka-cluster:WriteData` only to the `sensor-readings` topic, following the principle of least privilege
    - **If asked more:**
       - I would walk through a sample policy and explain how I tested it using IAM Policy Simulator before applying it to production roles
15. What is VPC?
    - **Answer:**
       - VPC (Virtual Private Cloud) is a logically isolated network within AWS
       - Our Kafka brokers and Spring Boot services ran inside a VPC with private subnets, so they were not accessible from the public internet
       - Only the Lambda function had access through a VPC endpoint
    - **If asked more:**
       - I would explain the VPC components — subnets, route tables, NAT gateways — and how we set up a public subnet for the load balancer and private subnets for the application tier
16. What is subnet?
    - **Answer:**
       - A subnet is a range of IP addresses within a VPC
       - In cold-chain, we used public subnets for the load balancer and private subnets for EC2 instances running Kafka and Spring Boot
       - The private subnets had no direct internet access, adding a security layer
    - **If asked more:**
       - I would explain the difference between public and private subnets — public subnets have a route to the internet gateway, private subnets route through NAT for outbound access only
17. What is security group?
    - **Answer:**
       - A security group acts as a virtual firewall for EC2 instances
       - I configured security groups to allow inbound traffic on port 9092 only from the Spring Boot application's security group, and port 8080 only from the load balancer
       - This micro-segmentation prevented direct access to services
    - **If asked more:**
       - I would explain stateful vs stateless firewalls — security groups are stateful, so return traffic is automatically allowed, unlike NACLs which are stateless
18. What is load balancer?
    - **Answer:**
       - A load balancer distributes incoming traffic across multiple EC2 instances for high availability and fault tolerance
       - In CDMS, we used an Application Load Balancer in front of the Spring Boot API instances to handle request routing, health checks, and SSL termination
    - **If asked more:**
       - I would explain how we configured the target groups, health check endpoints, and stickiness — we didn't need session stickiness because JWT tokens carried all session state
19. What is auto scaling?
    - **Answer:**
       - Auto Scaling automatically adjusts the number of EC2 instances based on demand
       - For the cold-chain Spring Boot API, I set a scale-out policy at 70% CPU and a scale-in policy at 30%, with a minimum of 2 and maximum of 6 instances to handle traffic spikes during temperature excursion events
    - **If asked more:**
       - I would explain the cooldown period and how we tested auto scaling with a load generator that simulated multiple dashboard users querying the API simultaneously
20. What is CloudWatch?
    - **Answer:**
       - CloudWatch is AWS's monitoring service for logs, metrics, and alarms
       - In the cold-chain project, I published custom metrics (consumer lag, processing rate, gateway status) to CloudWatch, set up dashboards, and configured alarms to email the team when temperature excursions were detected
    - **If asked more:**
       - I would explain how I used CloudWatch Logs to centralize logs from all EC2 instances and Lambda functions, and how I set up log group retention policies to save costs
21. How do you monitor AWS applications?
    - **Answer:**
       - I use CloudWatch for infrastructure metrics (CPU, memory, disk), custom application metrics published via the CloudWatch agent, and centralized logging with CloudWatch Logs
       - In cold-chain, I also configured detailed monitoring on EC2 and set up composite alarms that considered multiple metrics before paging the team
    - **If asked more:**
       - I would explain the difference between basic monitoring (5-minute intervals) and detailed monitoring (1-minute intervals), which I enabled for critical Kafka brokers
22. How do you store files in S3?
    - **Answer:**
       - I use the AWS SDK's `PutObjectRequest` to upload files from Spring Boot to S3
       - In CDMS, after the stored procedure generated the report CSV, the application uploaded it to an S3 bucket and returned a pre-signed URL that partners could use to download the report securely
    - **If asked more:**
       - I would explain pre-signed URLs — how I generated them with a 24-hour expiration, and how the bucket policy restricted access to only the application's IAM role
23. What is S3 bucket policy?
    - **Answer:**
       - An S3 bucket policy defines who can access the bucket and what operations they can perform
       - In CDMS, I wrote a bucket policy that allowed only the application's IAM role to upload files and allowed only authenticated partners (via pre-signed URL) to download specific objects
    - **If asked more:**
       - I would explain how I used bucket policies to enforce encryption in transit (AWS:SecureTransport) and deny public access at the bucket level
24. How do you secure S3?
    - **Answer:**
       - I secure S3 by blocking public access at the bucket level, using IAM policies for access control, enabling server-side encryption (SSE-S3), and using pre-signed URLs for temporary access
       - I also enabled S3 access logs to audit all requests and set up CloudWatch alarms for unusual access patterns
    - **If asked more:**
       - I would explain the shared responsibility model — AWS secures the infrastructure, I secure my buckets through proper configuration and access policies
25. How do you deploy Spring Boot on EC2?
    - **Answer:**
       - I build the Spring Boot application into a JAR using Maven, Dockerize it, push the image to Docker Hub, and Jenkins pulls the image on the EC2 instance and runs the container
       - The EC2 instance has the Docker runtime and environment variables configured via the user data script
    - **If asked more:**
       - I would explain the deployment script — how Jenkins connected to EC2 via SSH, stopped the old container, pulled the new image, and started the new container with health check verification
26. How do you run Kafka on EC2?
    - **Answer:**
       - I set up a 3-node Kafka cluster on EC2 instances, installed Kafka and ZooKeeper, configured `server.properties` with advertised listeners pointing to the private IPs, and set up replication factor 3
       - The instances were in private subnets with security groups allowing internal traffic on Kafka's port 9092
    - **If asked more:**
       - I would explain the challenges — handling ZooKeeper failure, configuring disks (we used EBS gp3 volumes), and tuning OS parameters (vm.swappiness, page cache) for Kafka's disk I/O pattern
27. What are the challenges of running Kafka on EC2?
    - **Answer:**
       - The main challenges are: (1) disk management — Kafka is I/O intensive, so EBS volume sizing and IOPS provisioning must be right; (2) network latency between brokers affects replication; (3) ZooKeeper adds operational complexity
       - In cold-chain, a broker once ran out of disk during a retention issue because a consumer wasn't keeping up
    - **If asked more:**
       - I would explain how I mitigated these by setting disk usage alerts, using multiple EBS volumes with RAID 0, and enabling Kafka's JBOD feature for cheaper storage
28. How do you manage environment variables in AWS?
    - **Answer:**
       - I use AWS Systems Manager Parameter Store to store environment-specific variables (database URLs, Kafka broker addresses)
       - The Spring Boot application reads these at startup using the AWS SDK
       - For secrets like passwords, I used AWS Secrets Manager with automatic rotation
    - **If asked more:**
       - I would explain the difference between Parameter Store (free, up to 10K parameters) and Secrets Manager (paid, automatic rotation), and why I chose Parameter Store for non-sensitive config and Secrets Manager for database credentials
29. How do you handle secrets?
    - **Answer:**
       - For production secrets (database passwords, API keys), I use AWS Secrets Manager
       - In CDMS, the Spring Boot application retrieved the database password from Secrets Manager at startup, cached it, and never stored it in code, properties files, or environment variables
    - **If asked more:**
       - I would explain the rotation process — Secrets Manager can automatically rotate RDS passwords using a Lambda function, and how the application handles stale connections after rotation
30. What is CI/CD deployment to AWS?
    - **Answer:**
       - CI/CD to AWS means automated build, test, and deployment pipelines using Jenkins
       - In our setup, Jenkins built the Spring Boot JAR, built a Docker image, published it to Docker Hub, and then deployed it to the EC2 instance by pulling the image and restarting the container
    - **If asked more:**
       - I would explain the full Jenkins pipeline stages: checkout, compile, test, build Docker image, push to registry, deploy to EC2, run health check, and rollback on failure
31. How did Jenkins deploy to EC2 in your project?
    - **Answer:**
       - Our Jenkins pipeline had stages for: (1) Maven build and run unit tests, (2) build Docker image with the Spring Boot JAR, (3) push image to Docker Hub, (4) SSH into the EC2 instance, (5) pull the new image, stop the old container, start the new one, and (6) run a health check curl command
    - **If asked more:**
       - I would explain how Jenkins credentials were stored securely — Docker Hub credentials and EC2 SSH keys were managed as Jenkins credentials and never exposed in the pipeline script
32. How do you debug AWS deployment failure?
    - **Answer:**
       - I start by checking the Jenkins build output for the specific error — compilation failure, Docker build failure, or deployment script error
       - Then I check the EC2 instance's CloudWatch logs for the application startup
       - Common issues: wrong environment variables, missing IAM permissions, or insufficient disk space
    - **If asked more:**
       - I would share a specific debugging story — a deployment failed because the EC2 instance had run out of disk from old Docker images, so I added a cleanup step to the Jenkins pipeline
33. What is high availability?
    - **Answer:**
       - High availability means a system stays operational despite component failures
       - In cold-chain, I achieved HA by deploying Kafka across 3 availability zones, running Spring Boot on 2+ EC2 instances behind a load balancer, and using S3 and RDS multi-AZ for data storage
    - **If asked more:**
       - I would explain how Kafka's ISR and automatic leader election provided HA for the messaging layer, and how the load balancer's health checks routed traffic away from failed instances
34. What is fault tolerance?
    - **Answer:**
       - Fault tolerance means a system continues operating correctly after a failure
       - In cold-chain, Kafka's replication factor of 3 meant the system tolerated a single broker failure without data loss or service interruption
       - The load balancer's target group marked unhealthy instances and routed traffic to healthy ones
    - **If asked more:**
       - I would explain how fault tolerance differs from HA — fault tolerance implies zero data loss, while HA implies minimal downtime
       - Kafka's acks=all with min.insync.replicas=2 provided both
35. What is horizontal scaling?
    - **Answer:**
       - Horizontal scaling means adding more instances to handle increased load
       - In cold-chain, when Kafka consumer lag increased, I added more consumer instances (up to the partition count) and scaled the Spring Boot API by increasing the auto-scaling group's desired count
    - **If asked more:**
       - I would compare horizontal vs vertical scaling — horizontal is more costly but provides elasticity and fault tolerance, while vertical is simpler but has a ceiling
36. What is vertical scaling?
    - **Answer:**
       - Vertical scaling means increasing the capacity of existing instances (more CPU, memory)
       - I vertically scaled the Kafka broker instances from m5.large (2 vCPU, 8GB) to m5.xlarge (4 vCPU, 16GB) when CPU utilization was consistently above 80%, because adding Kafka brokers doesn't increase per-partition throughput
    - **If asked more:**
       - I would explain when vertical scaling makes sense — stateful services like databases and Kafka where horizontal scaling is complex — and its limitation: you can only go as high as the largest instance type
37. What is Databricks?
    - **Answer:**
       - Databricks is a unified analytics platform for big data processing and machine learning
       - In the cold-chain project, Databricks was used for predictive analytics on historical temperature data to forecast equipment failures and optimize cooling schedules
    - **If asked more:**
       - I would explain how Databricks consumed data from our Kafka topic (via the Kafka connector) and ran Spark jobs to train predictive models on temperature excursion patterns
38. How did Databricks help in predictive analytics?
    - **Answer:**
       - Databricks analyzed historical temperature patterns stored in InfluxDB to predict when a cooling unit was likely to fail
       - The model was trained on temperature fluctuation data and sent predictions back through Kafka to trigger preventive maintenance alerts before an excursion occurred
    - **If asked more:**
       - I would explain how the predictions reduced unplanned downtime — by alerting the operations team 2-3 hours before a predicted failure, they could service the equipment during planned windows
39. What is data lake?
    - **Answer:**
       - A data lake stores raw data in its native format, unlike a data warehouse which stores structured data
       - In the cold-chain project, raw IoT data was stored in S3 as a data lake before being processed by Databricks for analytics
       - This allowed us to reprocess historical data with new algorithms without data loss
    - **If asked more:**
       - I would explain the difference between data lake (S3 with Parquet files) and data warehouse (MSSQL tables) — the data lake stored all raw sensor readings, while MSSQL stored aggregated reports
40. What is cloud cost optimization?
    - **Answer:**
       - Cloud cost optimization means minimizing AWS spend without sacrificing performance
       - In cold-chain, I optimized costs by: (1) using reserved instances for the Kafka brokers (67% savings vs on-demand), (2) transitioning old S3 reports to Glacier, (3) auto-stopping non-production EC2 instances overnight, and (4) right-sizing underutilized instances
    - **If asked more:**
       - I would explain how I used Cost Explorer to identify the biggest spend categories, and how setting up budget alerts prevented bill surprises when the team ran extra test environments
