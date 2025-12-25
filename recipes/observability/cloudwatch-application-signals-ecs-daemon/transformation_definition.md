# Enable Amazon CloudWatch Application Signals for Python Django on ECS with Daemon Strategy

## Objective
Enable comprehensive end-to-end application performance monitoring for a Python Django application running on Amazon ECS Fargate by implementing Amazon CloudWatch Application Signals using the CloudWatch Agent daemon strategy, including automatic instrumentation with AWS Distro for OpenTelemetry (ADOT), trace collection, metrics generation, and service map visualization.

## Summary
This transformation configures a Django application and its ECS infrastructure to emit telemetry data to CloudWatch Application Signals. The implementation uses a daemon-based architecture where a CloudWatch Agent runs as a separate ECS service and collects OpenTelemetry traces and metrics from the Django application containers. The Django application is auto-instrumented using ADOT Python, which captures HTTP requests, database queries, and outbound service calls without code changes. The CloudWatch Agent daemon receives telemetry via OTLP protocol and forwards it to CloudWatch Application Signals for visualization and analysis. IAM permissions, task definitions, service discovery, and environment variables are configured to enable the complete monitoring pipeline.

## Entry Criteria
1. Application is a Python Django web application (Django 3.2 or higher)
2. Application is deployed on Amazon ECS using Fargate launch type
3. Infrastructure is defined using AWS CDK (TypeScript or Python)
4. Application uses standard Django middleware and request/response patterns
5. ECS cluster has Container Insights enabled
6. Application has network connectivity to CloudWatch Agent service via AWS Cloud Map or service discovery
7. Task execution role has permissions to pull container images from ECR
8. Application Python dependencies are managed via requirements.txt or similar

## Implementation Steps

### 1. Update Python Dependencies for OpenTelemetry
Add the AWS Distro for OpenTelemetry (ADOT) auto-instrumentation dependencies to `requirements.txt`:
```
aws-opentelemetry-distro>=0.4.0
opentelemetry-instrumentation-django>=0.43b0
opentelemetry-instrumentation-psycopg2>=0.43b0
opentelemetry-instrumentation-boto3>=0.43b0
opentelemetry-exporter-otlp-proto-http>=1.21.0
```

### 2. Create CloudWatch Agent Daemon Service in CDK
In your CDK stack file (e.g., `silva-photography-stack.ts`), create a CloudWatch Agent daemon service that will collect telemetry from all application containers:

```typescript
// CloudWatch Agent Log Group
const cwAgentLogGroup = new logs.LogGroup(this, 'CwAgentLogGroup', {
  logGroupName: '/ecs/ecs-cwagent-daemon',
  removalPolicy: cdk.RemovalPolicy.DESTROY,
  retention: logs.RetentionDays.ONE_WEEK,
});

// CloudWatch Agent Task Role
const cwAgentTaskRole = new iam.Role(this, 'CwAgentTaskRole', {
  assumedBy: new iam.ServicePrincipal('ecs-tasks.amazonaws.com'),
  managedPolicies: [
    iam.ManagedPolicy.fromAwsManagedPolicyName('CloudWatchAgentServerPolicy'),
  ],
  inlinePolicies: {
    ApplicationSignalsPolicy: new iam.PolicyDocument({
      statements: [
        new iam.PolicyStatement({
          effect: iam.Effect.ALLOW,
          actions: [
            'cloudwatch:PutMetricData',
            'logs:PutLogEvents',
            'logs:CreateLogStream',
            'xray:PutTraceSegments',
            'xray:PutTelemetryRecords',
          ],
          resources: ['*'],
        }),
      ],
    }),
  },
});

// CloudWatch Agent Task Definition
const cwAgentTaskDefinition = new ecs.FargateTaskDefinition(this, 'CwAgentTaskDefinition', {
  memoryLimitMiB: 512,
  cpu: 256,
  taskRole: cwAgentTaskRole,
});

// CloudWatch Agent Container with Application Signals Configuration
const cwAgentContainer = cwAgentTaskDefinition.addContainer('cloudwatch-agent', {
  image: ecs.ContainerImage.fromRegistry('public.ecr.aws/cloudwatch-agent/cloudwatch-agent:latest'),
  essential: true,
  memoryReservationMiB: 512,
  cpu: 256,
  portMappings: [
    {
      containerPort: 4316,  // OTLP HTTP receiver
      protocol: ecs.Protocol.TCP,
    },
    {
      containerPort: 4317,  // OTLP gRPC receiver
      protocol: ecs.Protocol.TCP,
    },
  ],
  environment: {
    CW_CONFIG_CONTENT: JSON.stringify({
      traces: {
        traces_collected: {
          application_signals: {
            enabled: true,
          },
        },
      },
      logs: {
        metrics_collected: {
          application_signals: {
            enabled: true,
          },
        },
      },
    }),
  },
  logging: ecs.LogDrivers.awsLogs({
    streamPrefix: 'cwagent-daemon',
    logGroup: cwAgentLogGroup,
  }),
});

// Create CloudWatch Agent Daemon Service
const cwAgentService = new ecs.FargateService(this, 'CwAgentDaemonService', {
  cluster,
  taskDefinition: cwAgentTaskDefinition,
  desiredCount: 1,
  serviceName: 'cloudwatch-agent-daemon',
  enableExecuteCommand: true,
});
```

### 3. Configure Service Discovery for CloudWatch Agent
Create an AWS Cloud Map namespace and register the CloudWatch Agent service for DNS-based discovery:

```typescript
// Create Private DNS Namespace
const namespace = new servicediscovery.PrivateDnsNamespace(this, 'AppNamespace', {
  name: 'silva-photography.local',
  vpc,
});

// Enable Service Discovery for CloudWatch Agent
cwAgentService.enableCloudMap({
  name: 'cloudwatch-agent',
  cloudMapNamespace: namespace,
  dnsRecordType: servicediscovery.DnsRecordType.A,
});
```

### 4. Update Application Task Definition with ADOT Auto-Instrumentation
Configure the Django application task definition with ADOT auto-instrumentation using an init container pattern:

```typescript
// Create Task Role with Application Signals Permissions
const taskRole = new iam.Role(this, 'AppTaskRole', {
  assumedBy: new iam.ServicePrincipal('ecs-tasks.amazonaws.com'),
  managedPolicies: [
    iam.ManagedPolicy.fromAwsManagedPolicyName('AWSXRayDaemonWriteAccess'),
    iam.ManagedPolicy.fromAwsManagedPolicyName('CloudWatchAgentServerPolicy'),
  ],
});

// Task Definition with Shared Volume for ADOT
const taskDefinition = new ecs.FargateTaskDefinition(this, 'AppTaskDefinition', {
  memoryLimitMiB: 1024,
  cpu: 512,
  taskRole,
  volumes: [
    {
      name: 'opentelemetry-auto-instrumentation-python',
    },
  ],
});

// Init Container: Copy ADOT Auto-Instrumentation Files
const initContainer = taskDefinition.addContainer('init', {
  image: ecs.ContainerImage.fromRegistry('public.ecr.aws/aws-observability/adot-autoinstrumentation-python:v0.12.0'),
  essential: false,
  memoryReservationMiB: 64,
  cpu: 32,
  command: ['cp', '-a', '/autoinstrumentation/.', '/otel-auto-instrumentation-python'],
  logging: ecs.LogDrivers.awsLogs({
    streamPrefix: 'init',
    logGroup,
  }),
});

initContainer.addMountPoints({
  sourceVolume: 'opentelemetry-auto-instrumentation-python',
  containerPath: '/otel-auto-instrumentation-python',
  readOnly: false,
});
```

### 5. Configure Django Application Container with OpenTelemetry Environment Variables
Add the main Django application container with all necessary OpenTelemetry configuration:

```typescript
const appContainer = taskDefinition.addContainer('django-app', {
  image: ecs.ContainerImage.fromAsset('../', {
    file: 'Dockerfile',
    platform: Platform.LINUX_AMD64,
  }),
  essential: true,
  memoryReservationMiB: 768,
  cpu: 384,
  portMappings: [
    {
      containerPort: 8000,
      protocol: ecs.Protocol.TCP,
    },
  ],
  environment: {
    // Django Settings
    DJANGO_SETTINGS_MODULE: 'silva_photography.settings.production',
    
    // OpenTelemetry Resource Attributes
    OTEL_RESOURCE_ATTRIBUTES: 'service.name=silva-photography,service.version=1.0.0,deployment.environment=production',
    
    // OpenTelemetry Python Configuration
    PYTHONPATH: '/otel-auto-instrumentation-python/opentelemetry/instrumentation/auto_instrumentation:/otel-auto-instrumentation-python',
    OTEL_PYTHON_DISTRO: 'aws_distro',
    OTEL_PYTHON_CONFIGURATOR: 'aws_configurator',
    
    // Enable Auto-Instrumentation for Django, Psycopg2, and Boto3
    OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED: 'true',
    
    // Traces Configuration (send to CloudWatch Agent daemon)
    OTEL_TRACES_EXPORTER: 'otlp',
    OTEL_EXPORTER_OTLP_PROTOCOL: 'http/protobuf',
    OTEL_EXPORTER_OTLP_TRACES_ENDPOINT: 'http://cloudwatch-agent.silva-photography.local:4316/v1/traces',
    
    // Metrics Configuration (disable default metrics, use Application Signals)
    OTEL_METRICS_EXPORTER: 'none',
    OTEL_AWS_APPLICATION_SIGNALS_ENABLED: 'true',
    OTEL_AWS_APPLICATION_SIGNALS_EXPORTER_ENDPOINT: 'http://cloudwatch-agent.silva-photography.local:4316/v1/metrics',
    
    // Logs Configuration (send to CloudWatch Logs)
    OTEL_LOGS_EXPORTER: 'none',
    
    // Propagators for distributed tracing
    OTEL_PROPAGATORS: 'tracecontext,baggage,xray',
  },
  secrets: {
    // Database credentials and Django secrets
  },
  logging: ecs.LogDrivers.awsLogs({
    streamPrefix: 'django',
    logGroup,
  }),
});

// Mount ADOT instrumentation files
appContainer.addMountPoints({
  sourceVolume: 'opentelemetry-auto-instrumentation-python',
  containerPath: '/otel-auto-instrumentation-python',
  readOnly: false,
});

// Ensure init container completes before app starts
appContainer.addContainerDependencies({
  container: initContainer,
  condition: ecs.ContainerDependencyCondition.SUCCESS,
});
```

### 6. Update Dockerfile to Use OpenTelemetry Auto-Instrumentation
Modify your application's Dockerfile to use the `opentelemetry-instrument` wrapper for the Django application:

```dockerfile
# Existing Dockerfile content...

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Use opentelemetry-instrument to auto-instrument Django
CMD ["opentelemetry-instrument", "gunicorn", "--bind", "0.0.0.0:8000", "silva_photography.wsgi:application"]
```

### 7. Configure Django Settings for OpenTelemetry (Optional Manual Instrumentation)
If additional manual instrumentation is needed, update Django settings to integrate OpenTelemetry middleware:

```python
# settings/production.py or settings/base.py

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    # OpenTelemetry middleware will be auto-added by auto-instrumentation
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

# Optional: Configure additional instrumentation settings
OTEL_INSTRUMENT_DJANGO_EXCLUDED_URLS = '/health/,/static/'
```

### 8. Verify Network Connectivity Between Services
Ensure the Django application service can communicate with the CloudWatch Agent daemon service:

```typescript
// Allow ingress to CloudWatch Agent from application tasks
cwAgentService.connections.allowFrom(
  fargateService.service,
  ec2.Port.tcp(4316),
  'Allow Django app to send traces to CloudWatch Agent'
);
```

### 9. Deploy and Verify CloudWatch Application Signals
Deploy the updated CDK stack and verify Application Signals configuration:

```bash
# Build and deploy CDK stack
cd infrastructure
npm run build
cdk deploy

# Verify CloudWatch Agent is running
aws ecs describe-services \
  --cluster <cluster-name> \
  --services cloudwatch-agent-daemon

# Generate test traffic to Django application
curl http://<load-balancer-dns>/

# View Application Signals in CloudWatch Console
# Navigate to: CloudWatch > Application Signals > Services
```

### 10. Configure CloudWatch Dashboards and Alarms
Create CloudWatch dashboards to monitor Application Signals metrics:

```typescript
// Add to CDK stack for automated dashboard creation
const dashboard = new cloudwatch.Dashboard(this, 'AppSignalsDashboard', {
  dashboardName: 'silva-photography-app-signals',
});

dashboard.addWidgets(
  new cloudwatch.GraphWidget({
    title: 'Service Latency',
    left: [
      new cloudwatch.Metric({
        namespace: 'AWS/ApplicationSignals',
        metricName: 'Latency',
        dimensionsMap: {
          Service: 'silva-photography',
        },
        statistic: 'Average',
      }),
    ],
  }),
  new cloudwatch.GraphWidget({
    title: 'Error Rate',
    left: [
      new cloudwatch.Metric({
        namespace: 'AWS/ApplicationSignals',
        metricName: 'Fault',
        dimensionsMap: {
          Service: 'silva-photography',
        },
        statistic: 'Sum',
      }),
    ],
  })
);
```

## Validation / Exit Criteria
1. CloudWatch Agent daemon service is running successfully in the ECS cluster with status RUNNING
2. Django application containers start without errors and the init container completes successfully
3. Application logs show OpenTelemetry initialization messages indicating successful instrumentation
4. CloudWatch Application Signals console displays the service 'silva-photography' in the service map
5. HTTP requests to the Django application generate traces visible in CloudWatch Service Lens
6. Application Signals metrics (latency, error rate, request count) are being published to CloudWatch
7. Service map in Application Signals shows connections between Django application, RDS database, and S3 service
8. Distributed traces capture complete request flows including database queries and AWS SDK calls
9. CloudWatch Logs contain structured trace data from both the application and CloudWatch Agent
10. Application health check endpoint responds successfully and load balancer shows healthy targets
11. No performance degradation observed after enabling instrumentation (latency increase less than 5%)
12. IAM permissions allow task role to write traces and metrics to CloudWatch Application Signals
