## CloudWatch Application Signals - ECS Daemon Strategy ##

Overview

This transformation automatically configures a Python Django application running on Amazon ECS Fargate to send telemetry data to Amazon CloudWatch Application Signals using the CloudWatch Agent daemon strategy with AWS Distro for OpenTelemetry (ADOT) auto-instrumentation.

What This Transformation Does

Application Layer:
• Adds ADOT Python dependencies for automatic instrumentation
• Configures OpenTelemetry to capture HTTP requests, database queries, and AWS SDK calls
• Updates Dockerfile to use opentelemetry-instrument wrapper

Infrastructure Layer:
• Creates a CloudWatch Agent daemon service in ECS
• Sets up AWS Cloud Map for service discovery
• Configures IAM roles and permissions for Application Signals
• Updates task definitions with init container pattern for ADOT injection

Observability Pipeline:
• Enables distributed tracing with X-Ray propagation
• Configures OTLP endpoints for trace and metric collection
• Generates Application Signals metrics (latency, errors, throughput)
• Creates service maps showing application dependencies

Use Case

Perfect for teams who:
• Want end-to-end observability without manual instrumentation
• Need to monitor Django applications on ECS Fargate
• Want to see service maps, traces, and performance metrics
• Have limited time for observability projects

Solves these problems:
• Manual OpenTelemetry configuration is time-consuming
• Setting up CloudWatch Agent requires infrastructure expertise
• IAM permissions and networking are complex to configure
• Testing and validation takes significant effort

Prerequisites

Before running this transformation, ensure:
Application Requirements:
• Python Django 3.2 or higher
• Dependencies managed via requirements.txt
• Standard Django middleware and request/response patterns
Infrastructure Requirements:
• Application deployed on Amazon ECS with Fargate launch type
• Infrastructure defined using AWS CDK (TypeScript or Python)
• ECS cluster has Container Insights enabled
• Task execution role can pull images from ECR
AWS Permissions:
• CDK deployment permissions for your AWS account
• Ability to create IAM roles and policies
• CloudWatch and X-Ray write permissions
Network Configuration:
• VPC with private subnets for ECS tasks
• Network connectivity between application and CloudWatch Agent

Quick Start

What Gets Changed

Files Modified:
• requirements.txt - ADOT dependencies added
• Dockerfile - CMD updated to use opentelemetry-instrument
• CDK stack file (e.g., silva-photography-stack.ts) - CloudWatch Agent daemon, service discovery, and task definitions
• Django settings (optional) - OpenTelemetry middleware configuration

AWS Resources Created:
• CloudWatch Agent ECS service (daemon)
• AWS Cloud Map private DNS namespace
• IAM roles for Application Signals permissions
• CloudWatch Log Groups for agent and application
• Security group rules for OTLP traffic

Environment Variables Added:
• OTELRESOURCEATTRIBUTES - Service identification
• OTELTRACESEXPORTER - Trace export configuration
• OTELEXPORTEROTLPTRACESENDPOINT - CloudWatch Agent endpoint
• OTELAWSAPPLICATIONSIGNALSENABLED - Enable Application Signals
• OTEL_PROPAGATORS - Distributed tracing context propagation

Cost Estimate

Transformation Cost: ~$0.70 (based on real-world testing)
Time Required: ~20 minutes

Ongoing Costs (monthly estimates):
• CloudWatch Agent daemon: ~$15-20 (512 MB, 0.25 vCPU Fargate task)
• Application Signals metrics: ~$10-30 (depends on request volume)
• CloudWatch Logs: ~$5-15 (depends on log volume)
• X-Ray traces: ~$5-20 (depends on trace sampling rate)

Total estimated monthly cost: $35-85 (varies by application traffic)

Validation Steps

After the transformation completes, verify everything is working:
Check CloudWatch Agent Service

Expected: ACTIVE with runningCount: 1
Verify Application Containers Started

Check that tasks are in RUNNING state
Check Application Logs
Look for OpenTelemetry initialization messages:

Expected: Messages indicating ADOT instrumentation loaded successfully
View Application Signals
Open AWS Console → CloudWatch → Application Signals
Navigate to "Services" tab
Verify your service appears in the list
Click on service name to see service map
Generate Test Traffic
Check Service Map
In CloudWatch Application Signals:
• Service map should show your Django application
• Connections to RDS database should be visible
• Connections to S3 (if used) should appear
• Click on connections to see latency and error rates
View Distributed Traces
CloudWatch → X-Ray → Service Map
Click on your service node
Select "View traces"
Verify traces show complete request flows
Verify Metrics
CloudWatch → Application Signals → Metrics:
• Latency (p50, p90, p99)
• Error rate
• Request count
• Fault rate

Success Criteria

✅ CloudWatch Agent daemon service is RUNNING
✅ Django containers start without errors
✅ Init container completes successfully
✅ Service appears in Application Signals console
✅ Traces visible in CloudWatch Service Lens
✅ Metrics being published (latency, errors, requests)
✅ Service map shows application dependencies
✅ No performance degradation (< 5% latency increase)

Troubleshooting

Issue: CloudWatch Agent service won't start
• Check IAM permissions for task role
• Verify security group allows traffic on ports 4316/4317
• Review CloudWatch Agent logs for errors

Issue: No traces appearing in Application Signals
• Verify OTELEXPORTEROTLPTRACESENDPOINT points to correct DNS name
• Check network connectivity between app and agent
• Ensure service discovery is configured correctly

Issue: Application containers failing to start
• Check init container completed successfully
• Verify ADOT dependencies installed in requirements.txt
• Review application logs for Python import errors

Issue: High latency after instrumentation
• Adjust trace sampling rate: OTELTRACESSAMPLER=parentbased_traceidratio
• Set sampling ratio: OTELTRACESSAMPLER_ARG=0.1 (10% sampling)
• Review instrumented libraries and disable unnecessary ones

Related Recipes
• OpenTelemetry Lambda Instrumentation (Python) - Similar approach for Lambda functions
• X-Ray Instrumentation for Node.js - Alternative for Node.js applications
• CloudWatch Container Insights - Additional container-level metrics
• Multi-Service Tracing - Distributed tracing across microservices

Learn More
• AWS Transform Custom Documentation
• CloudWatch Application Signals Guide
• AWS Distro for OpenTelemetry
• OpenTelemetry Python Auto-Instrumentation

Transformation Definition Version: 1.0
Last Updated: December 2024
Tested With: Django 4.2, Python 3.11, ECS Fargate 1.4.0, ADOT Python 0.12.0

Author: Based on real-world implementation documented at builder.aws.com

I've created a comprehensive README for the transformation definition file! This README:

Explains what the transformation does at both application and infrastructure layers

Provides clear use cases and problems it solves

Lists all prerequisites so users know what's needed before running

Includes a quick start guide with the exact command to run

Details what gets changed (files, resources, environment variables)

Provides cost transparency with real estimates from your experience

Offers step-by-step validation with actual AWS CLI commands

Includes troubleshooting for common issues

Links to related recipes and learning resources

This README will help users understand exactly what the transformation does, whether it's right for their use case, and how to validate it worked correctly. Ready to add to your repository!

Would you like me to also create the prerequisites.md and validation.md files as separate, more detailed documents?
