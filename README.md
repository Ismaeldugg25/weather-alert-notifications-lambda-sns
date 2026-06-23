<img width="1280" height="720" alt="maxresdefault" src="https://github.com/user-attachments/assets/1b3ca223-a9e5-4b10-aadb-5eb052b10607" />

# Weather Alert Notifications System with Lambda and SNS

## Project Overview

This configuration creates a serverless weather monitoring system using AWS Lambda, SNS, EventBridge, and supporting services.

**Why this matters in the real world:** This exact pattern — *scheduled check → evaluate condition → notify* — is used everywhere in DevOps: health checks, cost anomaly alerts, certificate expiry warnings, disk space monitoring, etc. Once you understand this skeleton, you can swap "check weather" for "check anything.

## Problem

Organizations need timely weather alerts to protect outdoor operations, events, and employee safety, but manually monitoring weather conditions throughout the day is inefficient and prone to human error. Traditional monitoring solutions require dedicated infrastructure and maintenance overhead, while weather services don't provide customizable threshold-based alerting for specific business requirements. Without automated weather monitoring, businesses risk operational disruptions and safety incidents during severe weather conditions.

## Solution

Create a serverless weather monitoring system using AWS Lambda to periodically check weather conditions via a public API and Amazon SNS to deliver instant notifications when thresholds are exceeded. EventBridge provides reliable scheduling to trigger weather checks every hour, while Lambda's serverless architecture eliminates infrastructure management. This solution automatically scales based on demand, provides cost-effective monitoring, and delivers immediate alerts through multiple notification channels including email and SMS.. 

## Architecture Diagram

<img width="799" height="556" alt="image" src="https://github.com/user-attachments/assets/c61b160a-c4a7-4752-a709-76a7483c4afa" />


#### Breakdown of what is happening here:

1. **Scheduled Event** — The EventBridge Rule fires automatically every hour, triggering the Lambda Function.
2. **API Request** — The Lambda Function calls the external Weather API to get current conditions.
3. **Weather Data** — The Weather API returns the data back to Lambda.
4. **Alert Message** — If conditions exceed your thresholds, Lambda publishes a message to the SNS Topic.
5. **Email Alert** — SNS delivers the message to the Email Subscriber.
6. **SMS Alert** — SNS simultaneously fans out the same message to an SMS Subscriber (this is shown as a possible channel — your actual deployment only used email).

Running alongside this, in the background: the **IAM Role** grants Lambda permission to publish to SNS and write logs. 

## Prerequisites

- AWS account with permissions to create Lambda functions, SNS topics, EventBridge rules, and IAM roles
- Terraform (version 1.0 or later) installed
- AWS CLI installed and configured
- `pip3` installed locally (used by Terraform to bundle the `requests` dependency into the Lambda package)
- Free OpenWeatherMap API key from openweathermap.org (optional — project supports a demo mode using mock data, requiring no key)
- Estimated cost: $0.05–$0.20 per month for low-volume hourly usage (Lambda requests + SNS messages)

## Tools & Services Used

- **AWS Lambda** — serverless compute running the Python weather-check logic
- **Amazon SNS (Simple Notification Service)** — pub/sub messaging for fan-out email alerts
- **Amazon EventBridge** — scheduled trigger (hourly) for the Lambda function
- **Amazon CloudWatch** — log group for Lambda execution logs, plus alarms for errors and duration
- **AWS IAM** — execution role and policies for Lambda (least-privilege access to SNS/logs)
- **AWS KMS** — encryption at rest for the SNS topic (AWS-managed key)
- **Terraform** — infrastructure as code for the entire deployment
- **OpenWeatherMap API** — external weather data source (with a built-in demo/mock mode)
- **Python 3.12** — Lambda runtime language

## Preparation

The project's Terraform configuration is split across four files, each with a distinct responsibility:

- **`versions.tf`** — declares the required Terraform version and providers (`aws`, `random`, `archive`, and later `null`), and configures the AWS provider with `default_tags` applied to every resource
- **`variables.tf`** — declares all configurable inputs (region, environment, city, temperature/wind thresholds, schedule, notification email, Lambda sizing, API key) with validation rules on each
- **`main.tf`** — defines every AWS resource: IAM role, SNS topic, Lambda function, EventBridge rule, and CloudWatch alarms
- **`outputs.tf`** — defines what Terraform prints back after deployment (resource names, ARNs, console links, deployment summary)

## Steps

### **1. Define Terraform and Provider Requirements:**

Set the minimum Terraform version and pinned provider versions (`aws ~> 5.0`, `random ~> 3.6`, `archive ~> 2.4`) using pessimistic version constraints to avoid unexpected breaking changes from major version upgrades. Configured the AWS provider with `default_tags` so every resource created is automatically tagged with `Project`, `Environment`, and `ManagedBy = terraform` without repeating tags on every resource block.

<img width="549" height="584" alt="image-2" src="https://github.com/user-attachments/assets/5496ce0a-0a45-437b-8fd7-7d24a2561039" />


### **2. Define Input Variables:**

Declared all configurable inputs as Terraform variables rather than hardcoding values — region, environment, project name, city, temperature/wind thresholds, OpenWeatherMap API key (marked `sensitive` so it never prints in plan/apply output), notification email, EventBridge schedule expression, and Lambda memory/timeout. Each variable includes a `validation` block to catch invalid input (out-of-range thresholds, malformed email, invalid region format) before any AWS resources are touched.

<img width="650" height="383" alt="image-3" src="https://github.com/user-attachments/assets/17a748af-f075-4b91-9316-fc10e6e3b920" />

<img width="725" height="511" alt="image-4" src="https://github.com/user-attachments/assets/c5a39fb5-3f22-489e-b66d-b9d671979628" />

### **3. Create IAM Role for Lambda Function:**

AWS Lambda requires an execution role with permissions to write logs and publish to SNS. Created an IAM role with a trust policy allowing the `lambda.amazonaws.com` service to assume it, then attached the AWS-managed `AWSLambdaBasicExecutionRole` policy for CloudWatch logging, plus a custom inline policy granting `sns:Publish`scoped to only the specific SNS topic this project creates, following least-privilege practice.

<img width="620" height="402" alt="image-5" src="https://github.com/user-attachments/assets/146a02f6-27b6-4be7-a386-8174a0946694" />

<img width="677" height="433" alt="image-6" src="https://github.com/user-attachments/assets/42cef4f2-f636-4230-8726-a20a5878bd76" />


### **4. Create SNS Topic for Weather Notifications:**

Amazon SNS provides a managed messaging service for fan-out delivery to multiple endpoints. Created an SNS topic with server-side encryption enabled via the AWS-managed KMS key (`alias/aws/sns`), establishing the central hub all weather alerts and CloudWatch alarm notifications publish through.

<img width="660" height="274" alt="image-7" src="https://github.com/user-attachments/assets/2a090835-e5da-4166-ad13-23a110ab7439" />

### **5. Create SNS Topic Policy to Allow Lambda to Publish:**

Added a resource-based policy directly on the SNS topic, explicitly allowing the Lambda execution role to call `sns:Publish` against it. This sits alongside the identity-based policy on the IAM role itself, together they form a dual-authorization pattern common across AWS services where both the caller's permissions and the resource's policy must agree.

<img width="572" height="382" alt="image-8" src="https://github.com/user-attachments/assets/ad4121ed-4e1b-413a-9f3a-2da46fa6f707" />


### **6. Create Email Subscription for Notifications:**

Added an SNS topic subscription with the `email` protocol, pointing at the notification email provided as a variable. Used a conditional `count = var.notification_email != "" ? 1 : 0` so the subscription is only created if an email was actually supplied, keeping the deployment usable even without notification configured yet. After deployment, AWS automatically sent a confirmation email which had to be manually confirmed before any notifications could be delivered.

<img width="584" height="147" alt="image-9" src="https://github.com/user-attachments/assets/822572d8-9b05-48e7-9c78-fbbbb0cd1e51" />

### **7. Create Lambda Function Code:**

The Lambda function implements the core weather monitoring logic: fetch current conditions (or use built-in mock/demo data if no API key is provided), compare temperature and wind speed against configurable thresholds, and publish an SNS notification if either threshold is exceeded. The function also publishes error notifications on API failures or unexpected exceptions, so monitoring failures are never silent. The code was defined directly inside Terraform using a `local_file` resource, which writes the Python source to disk as part of `apply`.

### **8. Package and Deploy Lambda Function**:

AWS Lambda requires function code to be packaged as a ZIP file for deployment.

<img width="634" height="399" alt="image-10" src="https://github.com/user-attachments/assets/b1922c94-1469-417d-ac13-7bfe73dd6d73" />


### **9. Create EventBridge Scheduled Rule:**

Amazon EventBridge provides reliable, serverless scheduling. Created an `aws_cloudwatch_event_rule`(EventBridge's underlying Terraform resource name) with `schedule_expression = "rate(1 hour)"`, establishing the hourly trigger for weather checks without any infrastructure to maintain.

<img width="672" height="275" alt="image-11" src="https://github.com/user-attachments/assets/882a2fbc-9d4e-4983-90f9-8fe631af45e2" />


**Configure Lambda Permission and Target:**

EventBridge requires explicit permission to invoke a Lambda function. Added an `aws_cloudwatch_event_target`linking the rule to the Lambda function, and an `aws_lambda_permission` resource granting the `events.amazonaws.com`principal the right to invoke it, the resource-based policy equivalent of the dual-authorization pattern seen earlier with SNS.

<img width="579" height="277" alt="image-12" src="https://github.com/user-attachments/assets/7f1f03c9-9b12-48fc-8b34-747f63400412" />

### **10. Create CloudWatch Log Group and Alarms:**

Created a `aws_cloudwatch_log_group` with 14-day retention to control log storage costs, explicitly rather than relying on Lambda's default of unlimited retention. Added two CloudWatch alarms. One watching for Lambda errors, one watching execution duration. Both configured to publish to the same SNS topic used for weather alerts, reusing the notification pipeline for operational health monitoring.

<img width="697" height="397" alt="image-13" src="https://github.com/user-attachments/assets/f0ba09f5-0f03-492e-8d72-eb137ae68e0c" />

<img width="672" height="397" alt="image-14" src="https://github.com/user-attachments/assets/2400fe74-85bc-4223-806e-53353f996c8d" />


### **11. Set Real Configuration Values:**

Created a `terraform.tfvars` file with the real notification email and other overrides, leaving `weather_api_key` on its default `demo_key` value to guarantee a triggered alert on the first test run.

## Validation & Testing

#### **1. Confirm Email Subscription:**

Checked inbox for the AWS SNS confirmation email and clicked "Confirm subscription". Required before any notification can be delivered.

<img width="545" height="255" alt="image-15" src="https://github.com/user-attachments/assets/04297f95-0fe0-49af-8e4d-f8fe4517481d" />


#### **2. Test Lambda Function Manually:**

```
aws lambda invoke \
    --function-name $(terraform output -raw lambda_function_name) \
    --payload '{}' \
    response.json

cat response.json | python3 -m json.tool
```

Expected output: `statusCode: 200` with a triggered alert, since demo mode data is hardcoded to exceed both thresholds.

<img width="665" height="479" alt="image-16" src="https://github.com/user-attachments/assets/195a14e4-4725-4fbb-a6be-c8fad4693737" />


### Links
