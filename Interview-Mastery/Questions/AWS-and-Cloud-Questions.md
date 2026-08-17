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
      - Chose EC2 over Lambda for Kafka because Kafka needs persistent storage, sustained throughput, and a 15-minute Lambda timeout is insufficient for continuous broker operation
      - IoT Core handled device authentication using X.509 certificates — each gateway had a unique certificate registered in the device registry, and IoT Core validated the certificate during MQTT connection
      - CloudWatch alarms monitored custom metrics like temperature readings, and when a reading exceeded the threshold, CloudWatch triggered an SNS notification that alerted the team via email
2. What is EC2?
   - **Answer:**
      - EC2 is AWS's virtual server service
      - We deployed our Spring Boot APIs and Kafka brokers on EC2 instances, configured security groups for network access, and used auto-scaling to handle load
      - For Kafka, we chose EC2 over MSK to have full control over broker configuration
      - We used m5.large (2 vCPU, 8GB) for Spring Boot APIs where the workload was CPU-light, and m5.xlarge (4 vCPU, 16GB) for Kafka brokers because Kafka relies heavily on page cache and requires more memory for handling concurrent producer and consumer connections
3. What is S3?
   - **Answer:**
      - S3 is AWS's object storage service for storing and retrieving any amount of data
      - In CDMS, I used S3 to store generated reports — the optimized stored procedure output was written to S3 as CSV files that partners could download through the portal
      - We used S3 Standard for recent reports that partners accessed frequently, and transitioned reports older than 90 days to S3 Glacier using lifecycle rules to reduce storage costs
4. What is Lambda?
   - **Answer:**
      - Lambda is AWS's serverless compute service that runs code on demand
      - In the cold-chain pipeline, I used a Lambda function to transform MQTT messages from AWS IoT Core and publish them to Kafka
      - The Lambda was triggered by an IoT Core rule and ran for under a second per invocation
      - I configured the Lambda with 256MB memory because each MQTT message was small (under 1KB) and a 30-second timeout provided enough headroom without allowing stuck invocations. To mitigate cold starts, I used CloudWatch Events to invoke the Lambda every 5 minutes to keep it warm, ensuring consistent sub-second latency for real-time message processing
5. What is AWS IoT Core?
   - **Answer:**
      - AWS IoT Core is a managed cloud service that lets IoT devices connect and interact with AWS applications via MQTT
      - In cold-chain, IoT Core authenticated each gateway using device certificates, received temperature readings via MQTT, and routed them to Lambda through a rule
      - IoT Core's device shadows maintained the last reported state of each gateway, including temperature, battery level, and connectivity status. I set up a CloudWatch alarm that compared the shadow's timestamp against the current time, and when the gap exceeded 5 minutes, it triggered an alert indicating the gateway had gone offline unexpectedly
6. Why use AWS IoT Core?
   - **Answer:**
      - We used IoT Core because it handles the heavy lifting of MQTT broker management, device authentication via X.509 certificates, and scales automatically with the number of connected gateways
      - Setting up a custom MQTT broker on EC2 would have required more operational effort
      - Running a self-managed Mosquitto broker on EC2 would have required us to build device authentication, certificate management, and topic-based authorization from scratch. IoT Core provided a managed device registry with built-in X.509 certificate authentication and fine-grained topic-level policies, which eliminated the operational overhead of maintaining broker infrastructure and let us focus on the application layer
7. How does MQTT work with AWS IoT Core?
   - **Answer:**
      - IoT gateways establish persistent MQTT connections to AWS IoT Core using device certificates
      - The gateways publish temperature readings to a topic like `sensors/gateway1/temperature`, and IoT Core triggers a rule that forwards the message to Lambda for further processing
      - We used MQTT QoS 1 (at-least-once delivery) because temperature data needed reliable delivery without the overhead of QoS 2's exactly-once semantics. IoT Core's topic filters supported wildcard patterns like `sensors/+/temperature`, which allowed Lambda rules to subscribe to all gateway temperature readings regardless of gateway ID
8. Difference between EC2 and Lambda.
   - **Answer:**
      - EC2 provides full control over the OS and runtime, suitable for stateful or long-running services like Kafka
      - Lambda is ephemeral and event-driven, ideal for short stateless tasks
      - I chose EC2 for Kafka because it needs persistent storage and continuous operation, and Lambda for message transformation because it runs only when data arrives
      - Lambda's maximum 15-minute execution timeout and 10GB ephemeral storage limit make it unsuitable for Kafka, which requires continuous sustained throughput, persistent disk I/O, and long-running broker processes that operate 24/7 without interruption
9. When would you use Lambda?
   - **Answer:**
      - Use Lambda for event-driven, short-lived tasks like transforming data between services, resizing images, or responding to API Gateway requests
      - In cold-chain, Lambda was perfect for converting MQTT messages to Kafka-compatible JSON — a simple, stateless transformation that ran for milliseconds per event
      - Lambda is not a good fit for long-running processes exceeding 15 minutes, stateful processing that requires in-memory state across invocations, or high-throughput sustained workloads where per-invocation pricing becomes unpredictable — in those cases, EC2 or containers are more cost-effective
10. When would you avoid Lambda?
    - **Answer:**
       - Avoid Lambda for stateful services, long-running computations, or workloads requiring consistent low latency
       - In cold-chain, we avoided Lambda for the main data processing pipeline because we needed sustained processing of 1000+ msg/sec and Kafka consumers are better suited for that
       - Lambda can be more expensive than EC2 for high-utilization workloads because you pay per invocation and per GB-second of compute. Additionally, cold starts introduce 100-500ms of latency on initial invocation, which is unacceptable for real-time dashboards requiring consistent sub-100ms response times
11. What is serverless?
    - **Answer:**
       - Serverless means you don't manage servers — AWS handles scaling, patching, and availability
       - Lambda, IoT Core, and S3 are serverless services we used
       - Serverless is great for variable workloads because you pay only for what you use, but less suitable for predictable, high-utilization services like Kafka
       - Serverless reduces operational overhead by eliminating server management, patching, and capacity planning, but it can lead to unpredictable costs during traffic spikes and cold-start latency. We used a hybrid approach — serverless (Lambda + IoT Core) for variable-rate message ingestion, and EC2 for sustained high-throughput Kafka processing
12. What is IAM?
    - **Answer:**
       - IAM (Identity and Access Management) controls who can access AWS resources and what they can do
       - In cold-chain, I created IAM roles with least-privilege policies — Lambda had permission only to publish to the Kafka topic, and EC2 instances had access only to CloudWatch logs
       - IAM users represent human identities with long-term credentials (passwords, access keys), while IAM roles are assumed by AWS services with temporary credentials that rotate automatically. When I encountered access denied errors, I used CloudTrail to find the denied API call, identified the missing permission, and added it to the role's policy
13. What is IAM role?
    - **Answer:**
       - An IAM role is an identity that AWS services assume to get temporary permissions
       - In cold-chain, the Lambda function had an IAM role with policies that allowed it to receive messages from IoT Core and publish to the Kafka topic
       - This avoided hardcoding any credentials
       - The trust policy defined which service principal (e.g., `lambda.amazonaws.com`) was allowed to assume the role, and when Lambda executed, AWS automatically provided temporary credentials that rotated every few hours — no manual key management needed
14. What is IAM policy?
    - **Answer:**
       - An IAM policy defines specific permissions in JSON format
       - I wrote a policy that allowed the Lambda to `kafka-cluster:Connect` and `kafka-cluster:WriteData` only to the `sensor-readings` topic, following the principle of least privilege
       - A sample policy would look like `{"Effect": "Allow", "Action": ["kafka-cluster:Connect", "kafka-cluster:WriteData"], "Resource": "arn:aws:kafka:region:account:topic/sensor-readings"}`. Before applying any policy to production, I tested it in the IAM Policy Simulator to verify that the allowed and denied actions matched the intended behavior
15. What is VPC?
    - **Answer:**
       - VPC (Virtual Private Cloud) is a logically isolated network within AWS
       - Our Kafka brokers and Spring Boot services ran inside a VPC with private subnets, so they were not accessible from the public internet
       - Only the Lambda function had access through a VPC endpoint
       - A VPC consists of subnets (public and private), route tables defining traffic paths, an internet gateway for public internet access, and a NAT gateway for outbound internet access from private subnets. We deployed the load balancer in a public subnet (accessible from the internet) and all application EC2 instances in private subnets (no direct internet access)
16. What is subnet?
    - **Answer:**
       - A subnet is a range of IP addresses within a VPC
       - In cold-chain, we used public subnets for the load balancer and private subnets for EC2 instances running Kafka and Spring Boot
       - The private subnets had no direct internet access, adding a security layer
       - Public subnets have a route table entry pointing to an internet gateway, allowing resources to receive inbound internet traffic. Private subnets have no route to the internet gateway but can reach the internet outbound through a NAT gateway, which is used for pulling Docker images or accessing AWS services without exposing instances to inbound traffic
17. What is security group?
    - **Answer:**
       - A security group acts as a virtual firewall for EC2 instances
       - I configured security groups to allow inbound traffic on port 9092 only from the Spring Boot application's security group, and port 8080 only from the load balancer
       - This micro-segmentation prevented direct access to services
       - Security groups are stateful firewalls — when you allow inbound traffic, the return traffic is automatically allowed regardless of outbound rules. NACLs are stateless, meaning you must explicitly allow both inbound and outbound traffic. Security groups operate at the instance level, while NACLs operate at the subnet level
18. What is load balancer?
    - **Answer:**
       - A load balancer distributes incoming traffic across multiple EC2 instances for high availability and fault tolerance
       - In CDMS, we used an Application Load Balancer in front of the Spring Boot API instances to handle request routing, health checks, and SSL termination
       - We configured target groups pointing to the EC2 instances, with health checks hitting a `/health` endpoint that verified Spring Boot and database connectivity. We disabled session stickiness because JWT tokens carried all session state, making every request stateless and any instance capable of handling any request
19. What is auto scaling?
    - **Answer:**
       - Auto Scaling automatically adjusts the number of EC2 instances based on demand
       - For the cold-chain Spring Boot API, I set a scale-out policy at 70% CPU and a scale-in policy at 30%, with a minimum of 2 and maximum of 6 instances to handle traffic spikes during temperature excursion events
       - We set a cooldown period of 300 seconds to prevent rapid scale-out/scale-in oscillation. To test auto scaling, I used a load generator that simulated concurrent dashboard users querying the API, observed the CPU-based scale-out trigger, and verified that new instances joined the target group and started receiving traffic within 2-3 minutes
20. What is CloudWatch?
    - **Answer:**
       - CloudWatch is AWS's monitoring service for logs, metrics, and alarms
       - In the cold-chain project, I published custom metrics (consumer lag, processing rate, gateway status) to CloudWatch, set up dashboards, and configured alarms to email the team when temperature excursions were detected
       - I installed the CloudWatch agent on EC2 instances to stream application logs to CloudWatch Logs, and Lambda logs were automatically sent there. I set retention policies (30 days for debug logs, 90 days for application logs) to avoid unnecessary storage costs while keeping logs long enough for debugging
21. How do you monitor AWS applications?
    - **Answer:**
       - I use CloudWatch for infrastructure metrics (CPU, memory, disk), custom application metrics published via the CloudWatch agent, and centralized logging with CloudWatch Logs
       - In cold-chain, I also configured detailed monitoring on EC2 and set up composite alarms that considered multiple metrics before paging the team
       - Basic monitoring provides metrics at 5-minute intervals for EC2 instances, while detailed monitoring provides 1-minute intervals. I enabled detailed monitoring on Kafka brokers because consumer lag and throughput changes needed to be detected quickly to prevent data loss during traffic spikes
22. How do you store files in S3?
    - **Answer:**
       - I use the AWS SDK's `PutObjectRequest` to upload files from Spring Boot to S3
       - In CDMS, after the stored procedure generated the report CSV, the application uploaded it to an S3 bucket and returned a pre-signed URL that partners could use to download the report securely
       - Pre-signed URLs are temporary, time-limited URLs that grant direct access to S3 objects without requiring AWS credentials. I generated them with a 24-hour expiration using the Spring Boot application's IAM role, and the bucket policy was locked down to deny all direct access — only pre-signed URLs or the application's IAM role could read objects
23. What is S3 bucket policy?
    - **Answer:**
       - An S3 bucket policy defines who can access the bucket and what operations they can perform
       - In CDMS, I wrote a bucket policy that allowed only the application's IAM role to upload files and allowed only authenticated partners (via pre-signed URL) to download specific objects
       - I used a bucket policy condition `{"Bool": {"aws:SecureTransport": "false"}}` to deny any request that didn't use HTTPS, enforcing encryption in transit. I also added an explicit deny statement for any public access and blocked all public access settings at the bucket level
24. How do you secure S3?
    - **Answer:**
       - I secure S3 by blocking public access at the bucket level, using IAM policies for access control, enabling server-side encryption (SSE-S3), and using pre-signed URLs for temporary access
       - I also enabled S3 access logs to audit all requests and set up CloudWatch alarms for unusual access patterns
       - AWS is responsible for securing the underlying infrastructure (physical security, network, storage hardware), while I am responsible for securing my buckets through proper IAM policies, bucket policies, encryption settings, access logging, and blocking public access
25. How do you deploy Spring Boot on EC2?
    - **Answer:**
       - I build the Spring Boot application into a JAR using Maven, Dockerize it, push the image to Docker Hub, and Jenkins pulls the image on the EC2 instance and runs the container
       - The EC2 instance has the Docker runtime and environment variables configured via the user data script
       - The deployment script SSH'd into the EC2 instance using Jenkins-stored credentials, ran `docker stop` on the old container, `docker pull` to get the new image, `docker run` with the environment variables and volume mounts, and then executed a `curl` health check loop that waited for the `/health` endpoint to return 200 before marking the deployment as successful
26. How do you run Kafka on EC2?
    - **Answer:**
       - I set up a 3-node Kafka cluster on EC2 instances, installed Kafka and ZooKeeper, configured `server.properties` with advertised listeners pointing to the private IPs, and set up replication factor 3
       - The instances were in private subnets with security groups allowing internal traffic on Kafka's port 9092
       - Key challenges included ZooKeeper failure handling (configured a 3-node ZooKeeper ensemble with leader election), disk configuration (used EBS gp3 volumes with 3000 IOPS baseline for Kafka's write-heavy workload), and OS tuning (set vm.swappiness to 1 to minimize swap usage and increased the page cache size for Kafka's sequential I/O patterns)
27. What are the challenges of running Kafka on EC2?
    - **Answer:**
       - The main challenges are: (1) disk management — Kafka is I/O intensive, so EBS volume sizing and IOPS provisioning must be right; (2) network latency between brokers affects replication; (3) ZooKeeper adds operational complexity
       - In cold-chain, a broker once ran out of disk during a retention issue because a consumer wasn't keeping up
       - I mitigated disk issues by setting CloudWatch alarms at 70% and 85% disk usage, using multiple EBS volumes with software RAID 0 to increase IOPS beyond single-volume limits, and enabling Kafka's JBOD (Just a Bunch of Disks) feature to distribute partitions across multiple volumes for cost-effective storage scaling
28. How do you manage environment variables in AWS?
    - **Answer:**
       - I use AWS Systems Manager Parameter Store to store environment-specific variables (database URLs, Kafka broker addresses)
       - The Spring Boot application reads these at startup using the AWS SDK
       - For secrets like passwords, I used AWS Secrets Manager with automatic rotation
       - Parameter Store is free for up to 10,000 parameters and suitable for non-sensitive configuration like Kafka broker addresses and environment names. Secrets Manager costs per secret per month but provides automatic rotation via Lambda — I used it for database credentials where automatic password rotation was critical for security compliance
29. How do you handle secrets?
    - **Answer:**
       - For production secrets (database passwords, API keys), I use AWS Secrets Manager
       - In CDMS, the Spring Boot application retrieved the database password from Secrets Manager at startup, cached it, and never stored it in code, properties files, or environment variables
       - Secrets Manager rotation works by invoking a pre-configured Lambda function that generates a new password, updates it in RDS, and stores it in Secrets Manager in a single atomic operation. The Spring Boot application refreshed its cached secret on a configurable interval (e.g., every 10 minutes), so stale connections were drained and new ones used the rotated credentials
30. What is CI/CD deployment to AWS?
    - **Answer:**
       - CI/CD to AWS means automated build, test, and deployment pipelines using Jenkins
       - In our setup, Jenkins built the Spring Boot JAR, built a Docker image, published it to Docker Hub, and then deployed it to the EC2 instance by pulling the image and restarting the container
       - The full pipeline had stages: (1) checkout from Git, (2) Maven compile, (3) unit tests, (4) build Docker image tagged with the build number, (5) push to Docker Hub, (6) SSH deploy to EC2 pulling the new image, (7) health check endpoint verification, and (8) automatic rollback by restarting the previous container version if the health check failed
31. How did Jenkins deploy to EC2 in your project?
    - **Answer:**
       - Our Jenkins pipeline had stages for: (1) Maven build and run unit tests, (2) build Docker image with the Spring Boot JAR, (3) push image to Docker Hub, (4) SSH into the EC2 instance, (5) pull the new image, stop the old container, start the new one, and (6) run a health check curl command
       - Docker Hub credentials and EC2 SSH private keys were stored in Jenkins Credentials Store using the Credentials plugin, referenced by ID in the pipeline script (e.g., `withCredentials([usernamePassword(...)])`), and were never hardcoded or logged in the console output
32. How do you debug AWS deployment failure?
    - **Answer:**
       - I start by checking the Jenkins build output for the specific error — compilation failure, Docker build failure, or deployment script error
       - Then I check the EC2 instance's CloudWatch logs for the application startup
       - Common issues: wrong environment variables, missing IAM permissions, or insufficient disk space
       - A deployment once failed because the EC2 instance had run out of disk space from accumulated old Docker images. I diagnosed it by SSH'ing into the instance and running `df -h`, then added a `docker image prune -f` step to the Jenkins pipeline before pulling new images, which automatically cleaned up unused images after each deployment
33. What is high availability?
    - **Answer:**
       - High availability means a system stays operational despite component failures
       - In cold-chain, I achieved HA by deploying Kafka across 3 availability zones, running Spring Boot on 2+ EC2 instances behind a load balancer, and using S3 and RDS multi-AZ for data storage
       - Kafka's in-sync replicas (ISR) ensured that every partition had copies on multiple brokers, and if a broker failed, automatic leader election promoted an in-sync follower within seconds. The load balancer's health checks detected unhealthy instances within 30 seconds and stopped routing traffic to them until they recovered
34. What is fault tolerance?
    - **Answer:**
       - Fault tolerance means a system continues operating correctly after a failure
       - In cold-chain, Kafka's replication factor of 3 meant the system tolerated a single broker failure without data loss or service interruption
       - The load balancer's target group marked unhealthy instances and routed traffic to healthy ones
       - Fault tolerance means zero data loss during a failure (achieved with Kafka's `acks=all` and `min.insync.replicas=2`), while high availability means minimal downtime (achieved with multi-AZ deployment, load balancers, and auto-scaling). In cold-chain, both were critical — we couldn't lose temperature data (fault tolerance) and couldn't tolerate service outages (HA)
35. What is horizontal scaling?
    - **Answer:**
       - Horizontal scaling means adding more instances to handle increased load
       - In cold-chain, when Kafka consumer lag increased, I added more consumer instances (up to the partition count) and scaled the Spring Boot API by increasing the auto-scaling group's desired count
       - Horizontal scaling adds more instances, providing elasticity and fault tolerance (a single instance failure doesn't bring down the service), but it's more complex and costly for stateful systems. Vertical scaling increases the size of existing instances, which is simpler for stateful services like Kafka, but has a ceiling — you can only scale as large as the biggest available instance type
36. What is vertical scaling?
    - **Answer:**
       - Vertical scaling means increasing the capacity of existing instances (more CPU, memory)
       - I vertically scaled the Kafka broker instances from m5.large (2 vCPU, 8GB) to m5.xlarge (4 vCPU, 16GB) when CPU utilization was consistently above 80%, because adding Kafka brokers doesn't increase per-partition throughput
       - Vertical scaling makes sense for stateful services like databases and Kafka brokers, where horizontal scaling requires complex rebalancing of partitions and data replication. The key limitation is the ceiling — AWS instance types max out at certain sizes, and once you hit that limit, you must redesign the architecture to scale further
37. What is Databricks?
    - **Answer:**
       - Databricks is a unified analytics platform for big data processing and machine learning
       - In the cold-chain project, Databricks was used for predictive analytics on historical temperature data to forecast equipment failures and optimize cooling schedules
       - Databricks consumed data from our Kafka topic using the Kafka connector, reading raw temperature readings as they were published. It then ran Spark MLlib jobs to train predictive models on historical temperature excursion patterns, identifying early warning signals that preceded equipment failures
38. How did Databricks help in predictive analytics?
    - **Answer:**
       - Databricks analyzed historical temperature patterns stored in InfluxDB to predict when a cooling unit was likely to fail
       - The model was trained on temperature fluctuation data and sent predictions back through Kafka to trigger preventive maintenance alerts before an excursion occurred
       - The predictive model reduced unplanned downtime by sending maintenance alerts 2-3 hours before a predicted cooling unit failure. This allowed the operations team to schedule service during planned maintenance windows rather than responding to emergency calls at 2 AM when a refrigeration unit failed during transit
39. What is data lake?
    - **Answer:**
       - A data lake stores raw data in its native format, unlike a data warehouse which stores structured data
       - In the cold-chain project, raw IoT data was stored in S3 as a data lake before being processed by Databricks for analytics
       - This allowed us to reprocess historical data with new algorithms without data loss
       - The data lake (S3 with Parquet files) stored all raw IoT sensor readings in their native format, enabling reprocessing with new algorithms and serving as the single source of truth. The data warehouse (MSSQL tables) stored aggregated, structured reports optimized for fast query performance and partner-facing dashboards
40. What is cloud cost optimization?
    - **Answer:**
       - Cloud cost optimization means minimizing AWS spend without sacrificing performance
       - In cold-chain, I optimized costs by: (1) using reserved instances for the Kafka brokers (67% savings vs on-demand), (2) transitioning old S3 reports to Glacier, (3) auto-stopping non-production EC2 instances overnight, and (4) right-sizing underutilized instances
       - I used AWS Cost Explorer to analyze spending trends, which revealed that EC2 instances and EBS volumes were the biggest cost drivers. I set up monthly budget alerts at 80% and 100% thresholds that sent Slack and email notifications, preventing bill surprises when the team spun up extra test environments without informing the ops team
