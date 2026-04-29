# AWS — Beginner to Practical
## Complete Guide with Real Code Examples

> Covers all 12 core AWS services with working Java and CLI examples. Every topic includes architecture diagrams, code you can copy, and a real-world use case.

---

## Table of Contents

1. [EC2 — Virtual Servers](#1-ec2--virtual-servers)
2. [S3 — File Storage](#2-s3--file-storage)
3. [Lambda — Serverless Functions](#3-lambda--serverless-functions)
4. [DynamoDB — NoSQL Database](#4-dynamodb--nosql-database)
5. [SQS — Message Queue](#5-sqs--message-queue)
6. [SNS — Pub/Sub Notifications](#6-sns--pubsub-notifications)
7. [API Gateway — HTTP Endpoints](#7-api-gateway--http-endpoints)
8. [CloudWatch — Monitoring & Alerts](#8-cloudwatch--monitoring--alerts)
9. [IAM — Permissions & Security](#9-iam--permissions--security)
10. [CloudFormation — Infrastructure as Code](#10-cloudformation--infrastructure-as-code)
11. [RDS — Managed SQL Database](#11-rds--managed-sql-database)
12. [ELB + Auto Scaling — Scale to Any Traffic](#12-elb--auto-scaling--scale-to-any-traffic)

---

## 1. EC2 — Virtual Servers

**What it is:** EC2 (Elastic Compute Cloud) is a virtual server in AWS's data centre. You choose the OS, CPU, RAM, and storage. Your Spring Boot app runs on EC2 exactly like on your laptop — but it is always online.

> **Free tier:** t2.micro — 750 hours/month free for 12 months

### Architecture

```
Your browser → Internet → EC2 instance → Your Spring Boot app :8080
```

### Step 1 — Launch an EC2 instance (AWS CLI)

```bash
# Create a t2.micro EC2 instance running Amazon Linux 2
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t2.micro \
  --key-name my-key-pair \
  --security-group-ids sg-0123456789abcdef0 \
  --subnet-id subnet-0bb1c79de3EXAMPLE \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MySpringApp}]'

# Check the instance is running
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=MySpringApp" \
  --query 'Reservations[].Instances[].{State:State.Name,IP:PublicIpAddress}'
```

### Step 2 — Connect and deploy your Spring Boot app

```bash
# SSH into the server
ssh -i my-key-pair.pem ec2-user@<your-public-ip>

# Install Java 17 on the server
sudo yum update -y
sudo yum install java-17-amazon-corretto -y
java -version

# Upload your JAR from your local machine (run in a separate terminal)
scp -i my-key-pair.pem myapp.jar ec2-user@<your-ip>:/home/ec2-user/

# Start your Spring Boot app on the server
java -jar myapp.jar --server.port=8080 &

# Test it — replace with your EC2 public IP
curl http://<your-public-ip>:8080/api/health
```

> **Key concepts:** AMI = Amazon Machine Image (pre-built OS template). Security Group = firewall rules — open port 22 (SSH) and 8080 (your app). Key pair = SSH certificate to log in. t2.micro = 1 vCPU, 1 GB RAM — perfect for learning.

---

## 2. S3 — File Storage

**What it is:** S3 (Simple Storage Service) stores any file — images, PDFs, videos, CSVs, backups — at any scale. Think of it as a hard drive in the cloud that never fills up. Files are organised in "buckets."

> **Free tier:** 5 GB storage, 20,000 GET requests/month — always free

### Architecture

```
Spring Boot app → S3 bucket → profile-images/ | reports/ | backups/
```

### Maven dependency

```xml
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3</artifactId>
    <version>2.21.0</version>
</dependency>
```

### Java — upload, download, list, delete

```java
@Service
public class S3Service {

    private final S3Client s3 = S3Client.builder()
            .region(Region.US_EAST_1)
            .build();

    private static final String BUCKET = "my-app-bucket";

    // Upload a file to S3
    public String upload(String key, byte[] data) {
        s3.putObject(
            PutObjectRequest.builder()
                .bucket(BUCKET).key(key)
                .contentType("image/jpeg")
                .build(),
            RequestBody.fromBytes(data));
        return "https://" + BUCKET + ".s3.amazonaws.com/" + key;
    }

    // Download a file from S3
    public byte[] download(String key) {
        ResponseBytes<GetObjectResponse> obj = s3.getObjectAsBytes(
            GetObjectRequest.builder().bucket(BUCKET).key(key).build());
        return obj.asByteArray();
    }

    // List all files in S3 bucket
    public List<String> listFiles() {
        return s3.listObjectsV2(
            ListObjectsV2Request.builder().bucket(BUCKET).build())
            .contents().stream()
            .map(S3Object::key).toList();
    }

    // Delete a file from S3
    public void delete(String key) {
        s3.deleteObject(
            DeleteObjectRequest.builder().bucket(BUCKET).key(key).build());
    }
}
```

> **Real use case:** User uploads a profile photo → Spring Boot controller receives it → calls `s3Service.upload("profiles/user123.jpg", bytes)` → stores the returned URL in your database. Next time, serve the URL directly — S3 handles CDN-level delivery.

---

## 3. Lambda — Serverless Functions

**What it is:** Lambda runs your code only when triggered — no server to manage, no idle cost. You write a function, upload it, and AWS runs it in response to events: an HTTP request, a file uploaded to S3, a message in SQS, or a scheduled timer.

> **Free tier:** 1 million requests and 400,000 GB-seconds compute per month — always free

### Architecture

```
File uploaded to S3  →  triggers  →  Lambda function  →  Resize image + save back to S3
```

### Java Lambda handler — S3 trigger

```java
public class ImageResizeHandler
        implements RequestHandler<S3Event, String> {

    @Override
    public String handleRequest(S3Event event, Context context) {

        // Get the S3 object that triggered this Lambda
        String bucket = event.getRecords().get(0)
                .getS3().getBucket().getName();
        String key    = event.getRecords().get(0)
                .getS3().getObject().getKey();

        context.getLogger().log("Processing file: " + bucket + "/" + key);

        // Your business logic here — resize, transform, notify...
        processImage(bucket, key);

        return "Processed: " + key;
    }
}
```

### Java Lambda handler — API Gateway (HTTP trigger)

```java
public class HelloLambda
        implements RequestHandler<APIGatewayProxyRequestEvent,
                                    APIGatewayProxyResponseEvent> {

    @Override
    public APIGatewayProxyResponseEvent handleRequest(
            APIGatewayProxyRequestEvent request, Context ctx) {

        String name = request.getQueryStringParameters()
                          .getOrDefault("name", "World");
        return new APIGatewayProxyResponseEvent()
                .withStatusCode(200)
                .withBody("Hello, " + name + "!");
    }
}
```

### Deploy with AWS CLI

```bash
# Package your Lambda as a JAR
mvn package -DskipTests

# Create the Lambda function
aws lambda create-function \
  --function-name ImageResizer \
  --runtime java17 \
  --handler com.example.ImageResizeHandler::handleRequest \
  --role arn:aws:iam::123456789012:role/lambda-role \
  --zip-file fileb://target/myapp.jar

# Test it directly
aws lambda invoke \
  --function-name ImageResizer \
  --payload '{"name":"Suresh"}' response.json
```

> **When to use Lambda over EC2:** Use Lambda for short-lived, event-driven tasks — image resizing, sending emails, processing SQS messages, scheduled jobs. Use EC2 for long-running servers. Lambda is billed per 100ms of execution — zero cost when idle.

---

## 4. DynamoDB — NoSQL Database

**What it is:** DynamoDB is a fully managed NoSQL database — no servers to manage, automatic scaling, millisecond latency. Data is organised by a primary key. Perfect for session data, user profiles, IoT data, and high-throughput workloads.

> **Free tier:** 25 GB storage, 25 read/write capacity units — always free

### Architecture

```
Table: Users
  Partition key: userId (String)
  Attributes:    name, email, createdAt
```

### Java — full CRUD with AWS SDK v2

```java
@Service
public class DynamoUserService {

    private final DynamoDbClient db = DynamoDbClient.builder()
            .region(Region.US_EAST_1).build();

    private static final String TABLE = "Users";

    // CREATE — save a user
    public void saveUser(String userId, String name, String email) {
        db.putItem(PutItemRequest.builder()
            .tableName(TABLE)
            .item(Map.of(
                "userId",    AttributeValue.fromS(userId),
                "name",      AttributeValue.fromS(name),
                "email",     AttributeValue.fromS(email),
                "createdAt", AttributeValue.fromS(Instant.now().toString())))
            .build());
    }

    // READ — get a user by ID
    public Map<String, AttributeValue> getUser(String userId) {
        return db.getItem(GetItemRequest.builder()
            .tableName(TABLE)
            .key(Map.of("userId", AttributeValue.fromS(userId)))
            .build()).item();
    }

    // UPDATE — change a user's email
    public void updateEmail(String userId, String newEmail) {
        db.updateItem(UpdateItemRequest.builder()
            .tableName(TABLE)
            .key(Map.of("userId", AttributeValue.fromS(userId)))
            .updateExpression("SET email = :e")
            .expressionAttributeValues(Map.of(
                ":e", AttributeValue.fromS(newEmail)))
            .build());
    }

    // DELETE — remove a user
    public void deleteUser(String userId) {
        db.deleteItem(DeleteItemRequest.builder()
            .tableName(TABLE)
            .key(Map.of("userId", AttributeValue.fromS(userId)))
            .build());
    }
}
```

> **DynamoDB vs SQL:** No joins in DynamoDB — design your table around your query patterns. If you always look up users by userId, that is your partition key. For complex queries (filter by email), add a Global Secondary Index (GSI). DynamoDB scales to millions of requests per second automatically — MySQL cannot.

---

## 5. SQS — Message Queue

**What it is:** SQS (Simple Queue Service) is a message queue. Service A puts a message in the queue. Service B picks it up later. This decouples them — A does not need to wait for B, and if B is down, messages wait safely in the queue.

> **Free tier:** 1 million requests/month — always free

### Architecture

```
Order service  →  sends message  →  SQS Queue  →  picks up  →  Email service  →  Sends email
```

### Java — producer (send) and consumer (receive)

```java
@Service
public class SqsService {

    private final SqsClient sqs = SqsClient.builder()
            .region(Region.US_EAST_1).build();

    private static final String QUEUE_URL =
        "https://sqs.us-east-1.amazonaws.com/123456789/OrderQueue";

    // PRODUCER — send an order event to the queue
    public void sendOrderEvent(String orderId, String email) {
        String body = """
            { "orderId": "%s", "email": "%s", "status": "PLACED" }
            """.formatted(orderId, email);

        sqs.sendMessage(SendMessageRequest.builder()
            .queueUrl(QUEUE_URL)
            .messageBody(body)
            .delaySeconds(0)
            .build());
    }

    // CONSUMER — pick up messages and process them
    public void processMessages() {
        ReceiveMessageResponse response = sqs.receiveMessage(
            ReceiveMessageRequest.builder()
                .queueUrl(QUEUE_URL)
                .maxNumberOfMessages(10)   // up to 10 at once
                .waitTimeSeconds(20)        // long-poll — wait up to 20s
                .build());

        response.messages().forEach(msg -> {
            System.out.println("Processing: " + msg.body());

            // Delete message AFTER successful processing
            sqs.deleteMessage(DeleteMessageRequest.builder()
                .queueUrl(QUEUE_URL)
                .receiptHandle(msg.receiptHandle())
                .build());
        });
    }
}
```

> **Critical rule:** Always delete the message AFTER you have processed it successfully. If your consumer crashes mid-processing, the message becomes visible again after the visibility timeout and will be retried. Configure a Dead Letter Queue (DLQ) to capture messages that fail repeatedly.

---

## 6. SNS — Pub/Sub Notifications

**What it is:** SNS (Simple Notification Service) is a pub/sub system. You publish ONE message to a topic and SNS fans it out to ALL subscribers simultaneously — email addresses, SMS numbers, Lambda functions, SQS queues, HTTPS endpoints.

> **Free tier:** 1 million publishes/month, 1,000 email notifications — always free

### Architecture — fan-out pattern

```
Order placed  →  publish  →  SNS Topic
                                  ├── SQS: email queue
                                  ├── Lambda: analytics
                                  └── SMS: admin alert
```

### Java — publish to SNS topic

```java
@Service
public class SnsService {

    private final SnsClient sns = SnsClient.builder()
            .region(Region.US_EAST_1).build();

    private static final String TOPIC_ARN =
        "arn:aws:sns:us-east-1:123456789012:OrderEvents";

    // Publish an event — ALL subscribers receive this simultaneously
    public String publishOrderEvent(String orderId, String status) {
        String message = """
            {
              "orderId":   "%s",
              "status":    "%s",
              "timestamp": "%s"
            }""".formatted(orderId, status, Instant.now());

        PublishResponse response = sns.publish(
            PublishRequest.builder()
                .topicArn(TOPIC_ARN)
                .subject("Order Event: " + status)
                .message(message)
                .build());

        return response.messageId();
    }
}

// In your Order controller — trigger SNS when order is placed
@PostMapping("/orders")
public Order placeOrder(@RequestBody OrderRequest req) {
    Order order = orderService.create(req);
    snsService.publishOrderEvent(order.getId(), "PLACED");
    return order;
    // Email service, analytics Lambda, and admin SMS all receive this
    // simultaneously — no direct coupling between services
}
```

> **SNS vs SQS:** SNS = broadcast to many subscribers at once (fire and forget). SQS = one consumer processes each message (reliable queue with retry). In production, combine them — SNS fans out to multiple SQS queues. This is the SNS-SQS fan-out pattern.

---

## 7. API Gateway — HTTP Endpoints

**What it is:** API Gateway creates HTTP endpoints that route to your Lambda functions, EC2 apps, or other services. It handles auth, rate limiting, SSL, and CORS so your backend does not have to. Every public REST API needs a front door.

> **Free tier:** 1 million API calls/month for 12 months

### Architecture

```
Client app  →  API Gateway (HTTPS)  →  Lambda function  →  DynamoDB
```

### Create REST API via AWS CLI

```bash
# 1. Create the API
aws apigateway create-rest-api --name "UserAPI"
# Returns: { "id": "abc123def" }

# 2. Get the root resource ID
aws apigateway get-resources --rest-api-id abc123def

# 3. Create /users resource
aws apigateway create-resource \
  --rest-api-id abc123def \
  --parent-id <root-resource-id> \
  --path-part users

# 4. Add GET method to /users
aws apigateway put-method \
  --rest-api-id abc123def \
  --resource-id <users-resource-id> \
  --http-method GET \
  --authorization-type NONE

# 5. Connect to Lambda function
aws apigateway put-integration \
  --rest-api-id abc123def \
  --resource-id <users-resource-id> \
  --http-method GET \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:123:function:GetUsers/invocations

# 6. Deploy to a stage
aws apigateway create-deployment \
  --rest-api-id abc123def \
  --stage-name prod

# Your API is now live at:
# https://abc123def.execute-api.us-east-1.amazonaws.com/prod/users
```

> **Serverless pattern:** API Gateway + Lambda + DynamoDB = fully serverless REST API. Zero servers to manage. Pay only for actual requests. Scale to zero when idle. Scale to millions of requests automatically.

---

## 8. CloudWatch — Monitoring & Alerts

**What it is:** CloudWatch collects logs and metrics from every AWS service and your application. Set alarms that notify you via SNS when CPU exceeds 80% or error rate spikes. This is how you know your production app is healthy.

> **Free tier:** 10 custom metrics, 10 alarms, 5 GB logs — always free

### Java — send custom metrics and create alarms

```java
@Service
public class CloudWatchService {

    private final CloudWatchClient cw = CloudWatchClient.builder()
            .region(Region.US_EAST_1).build();

    // Send a custom metric — e.g. orders placed per minute
    public void recordOrderCount(int count) {
        cw.putMetricData(PutMetricDataRequest.builder()
            .namespace("MyApp/Orders")
            .metricData(MetricDatum.builder()
                .metricName("OrdersPlaced")
                .value((double) count)
                .unit(StandardUnit.COUNT)
                .timestamp(Instant.now())
                .build())
            .build());
    }

    // Create an alarm — notify via SNS when errors > 10 in 5 minutes
    public void createErrorAlarm(String snsTopicArn) {
        cw.putMetricAlarm(PutMetricAlarmRequest.builder()
            .alarmName("HighErrorRate")
            .metricName("Errors")
            .namespace("MyApp/Orders")
            .statistic(Statistic.SUM)
            .period(300)           // 5 minutes
            .evaluationPeriods(1)
            .threshold(10.0)       // alarm when > 10 errors
            .comparisonOperator(ComparisonOperator.GREATER_THAN_THRESHOLD)
            .alarmActions(snsTopicArn)
            .build());
    }
}
```

### Structured logging from Spring Boot

```yaml
# application.yml — logs automatically shipped to CloudWatch on EC2
logging:
  level:
    root: INFO
    com.example: DEBUG
```

```java
@Slf4j
@Service
public class OrderService {
    public Order process(OrderRequest req) {
        log.info("Processing order orderId={} userId={} amount={}",
                  req.getId(), req.getUserId(), req.getAmount());
        try {
            return placeOrder(req);
        } catch (Exception e) {
            log.error("Order failed orderId={} error={}", req.getId(), e.getMessage());
            throw e;
        }
    }
}
```

> **Production must-have:** On day one set up: (1) a CloudWatch alarm on error count, (2) a log group for each service, (3) a dashboard showing request rate, error rate, and latency. This is the difference between knowing your app is broken and finding out from a user.

---

## 9. IAM — Permissions & Security

**What it is:** IAM (Identity and Access Management) controls who can do what in your AWS account. Never give your application full admin access. Create a specific IAM Role with only the permissions it needs. This is the single most important AWS security concept.

> **IAM is always free — no usage cost**

### Principle of least privilege

```
Root account  →  create  →  IAM Role: LambdaRole  →  allows ONLY  →  DynamoDB:GetItem on Users table
```

### Create IAM policy + role via AWS CLI

```bash
# Step 1: Create a policy that allows S3 read + DynamoDB write only
aws iam create-policy \
  --policy-name MyAppPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["s3:GetObject", "s3:PutObject"],
        "Resource": "arn:aws:s3:::my-app-bucket/*"
      },
      {
        "Effect": "Allow",
        "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:UpdateItem"],
        "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/Users"
      },
      {
        "Effect": "Allow",
        "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
        "Resource": "*"
      }
    ]
  }'

# Step 2: Create an IAM Role for Lambda
aws iam create-role \
  --role-name MyLambdaRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Step 3: Attach the policy to the role
aws iam attach-role-policy \
  --role-name MyLambdaRole \
  --policy-arn arn:aws:iam::123456789012:policy/MyAppPolicy
```

> **Never do this:** Do not hardcode AWS access keys in your code (`accessKeyId`, `secretAccessKey`). Use IAM Roles — they provide temporary credentials automatically. If access keys leak to GitHub, bots scan for them within seconds and your AWS bill can hit thousands of dollars overnight.

---

## 10. CloudFormation — Infrastructure as Code

**What it is:** CloudFormation lets you define your entire AWS infrastructure in a YAML file and deploy it with one command. Instead of clicking through the AWS console, you write code that creates EC2, S3, DynamoDB, Lambda, and IAM — all at once, repeatably.

> **CloudFormation is free — you pay only for the resources it creates**

### Complete stack — Lambda + DynamoDB + IAM Role

```yaml
# stack.yaml — your entire infrastructure in one file
AWSTemplateFormatVersion: '2010-09-09'
Description: 'My App Stack — Lambda + DynamoDB + IAM'

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]

Resources:

  # DynamoDB table
  UsersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub 'Users-${Environment}'
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: userId
          AttributeType: S
      KeySchema:
        - AttributeName: userId
          KeyType: HASH

  # IAM Role for Lambda
  LambdaRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Statement:
          - Effect: Allow
            Principal: { Service: lambda.amazonaws.com }
            Action: 'sts:AssumeRole'
      Policies:
        - PolicyName: LambdaPolicy
          PolicyDocument:
            Statement:
              - Effect: Allow
                Action: [dynamodb:GetItem, dynamodb:PutItem]
                Resource: !GetAtt UsersTable.Arn

  # Lambda function
  UserFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub 'UserHandler-${Environment}'
      Runtime: java17
      Handler: com.example.UserHandler::handleRequest
      Role: !GetAtt LambdaRole.Arn
      Code:
        S3Bucket: my-deployment-bucket
        S3Key: myapp.jar
      Environment:
        Variables:
          TABLE_NAME: !Ref UsersTable

Outputs:
  TableName: { Value: !Ref UsersTable }
  FunctionArn: { Value: !GetAtt UserFunction.Arn }
```

### Deploy the stack

```bash
# Deploy for dev environment
aws cloudformation deploy \
  --template-file stack.yaml \
  --stack-name my-app-dev \
  --parameter-overrides Environment=dev \
  --capabilities CAPABILITY_IAM

# Deploy for prod — same template, different parameters
aws cloudformation deploy \
  --template-file stack.yaml \
  --stack-name my-app-prod \
  --parameter-overrides Environment=prod \
  --capabilities CAPABILITY_IAM

# Delete everything (useful for dev cleanup)
aws cloudformation delete-stack --stack-name my-app-dev
```

> **Why CloudFormation matters:** One command creates your entire environment identically every time — dev, staging, and prod are guaranteed to match. If production has a problem, you can tear it down and redeploy in minutes. Never manage AWS by clicking through the console manually.

---

## 11. RDS — Managed SQL Database

**What it is:** RDS (Relational Database Service) runs MySQL, PostgreSQL, Oracle, or SQL Server — fully managed. Automatic backups, patching, failover, and scaling. Your Spring Boot + JPA works with RDS exactly like a local database — just change the connection URL.

> **Free tier:** db.t3.micro MySQL/PostgreSQL — 750 hours/month for 12 months

### Architecture

```
Spring Boot app (EC2)  →  JDBC  →  RDS MySQL (private subnet)  →  automated backups to S3
```

### Create RDS instance via CLI

```bash
aws rds create-db-instance \
  --db-instance-identifier myapp-db \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.0 \
  --master-username admin \
  --master-user-password MySecurePass123! \
  --allocated-storage 20 \
  --backup-retention-period 7 \
  --no-publicly-accessible
  # Private — only accessible from within your VPC
```

### Spring Boot — connect to RDS

```yaml
# application.yml — point to your RDS endpoint
spring:
  datasource:
    url: jdbc:mysql://myapp-db.abc123.us-east-1.rds.amazonaws.com:3306/myappdb
    username: admin
    password: ${DB_PASSWORD}    # use environment variable — never hardcode!
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

```java
// Your JPA entities and repositories work unchanged — RDS is just MySQL
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}
```

> **RDS vs DynamoDB:** Use RDS when you have complex relational data with joins, transactions, and SQL queries. Use DynamoDB for simple key-value lookups at massive scale. Most enterprise apps use both — RDS for core business data, DynamoDB for session or cache data.

---

## 12. ELB + Auto Scaling — Scale to Any Traffic

**What it is:** Elastic Load Balancer distributes incoming traffic across multiple EC2 instances. Auto Scaling Group automatically adds instances when traffic spikes and removes them when it drops. Together they make your app scale to any load with zero downtime.

> **Free tier:** 750 hours ELB/month for 12 months. Auto Scaling is free — you pay for EC2 instances.

### Architecture

```
10,000 users  →  Application Load Balancer
                        ├── EC2: app-1
                        ├── EC2: app-2       →  RDS
                        └── EC2: app-3 (auto-added when CPU > 70%)
```

### Set up Load Balancer + Auto Scaling via CLI

```bash
# Step 1: Create Application Load Balancer
aws elbv2 create-load-balancer \
  --name myapp-alb \
  --subnets subnet-aaa subnet-bbb \
  --security-groups sg-0123456789 \
  --type application

# Step 2: Create a target group (where traffic is sent)
aws elbv2 create-target-group \
  --name myapp-targets \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-0123456 \
  --health-check-path /api/health    # Spring Boot health endpoint

# Step 3: Create Launch Template (defines EC2 instance configuration)
aws ec2 create-launch-template \
  --launch-template-name myapp-template \
  --launch-template-data '{
    "ImageId": "ami-0c55b159cbfafe1f0",
    "InstanceType": "t3.small",
    "KeyName": "my-key-pair"
  }'

# Step 4: Create Auto Scaling Group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name myapp-asg \
  --launch-template LaunchTemplateName=myapp-template,Version='$Latest' \
  --min-size 2 \           # always at least 2 instances (high availability)
  --max-size 10 \          # never more than 10
  --desired-capacity 2 \   # start with 2
  --vpc-zone-identifier "subnet-aaa,subnet-bbb" \
  --target-group-arns arn:aws:elasticloadbalancing:...

# Step 5: Add scaling policy — scale up when CPU > 70%
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name myapp-asg \
  --policy-name ScaleOnCPU \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 70.0
  }'
```

> **How it works end to end:** Traffic arrives at the Load Balancer DNS. ALB health checks confirm each EC2 is healthy via `/api/health`. If CPU across the group hits 70%, Auto Scaling adds a new EC2, installs your app, and the ALB starts routing to it within 3–5 minutes — automatically. When traffic drops, it terminates extra instances to save cost.

---

## Quick Reference — All 12 Services

| Service | What it does | Free tier |
|---|---|---|
| **EC2** | Virtual server — run any app | 750 hrs/month (12 months) |
| **S3** | File storage — images, backups, CSVs | 5 GB always free |
| **Lambda** | Run code without a server | 1M requests always free |
| **DynamoDB** | NoSQL database at any scale | 25 GB always free |
| **SQS** | Message queue — decouple services | 1M requests always free |
| **SNS** | Pub/sub — notify many subscribers | 1M publishes always free |
| **API Gateway** | HTTP front door to your backend | 1M calls/month (12 months) |
| **CloudWatch** | Monitoring, logs, and alerts | 10 metrics + 5 GB logs free |
| **IAM** | Permissions and security | Always free |
| **CloudFormation** | Infrastructure as code | Always free |
| **RDS** | Managed MySQL / PostgreSQL | 750 hrs/month (12 months) |
| **ELB + Auto Scaling** | Load balancing + auto-scale | 750 hrs ELB (12 months) |

---

*All examples use the AWS SDK for Java v2 and AWS CLI v2. Free tier limits are as of 2025 — verify current limits at aws.amazon.com/free.*
