# Platform Architecture

## Overview

The NStarX Data Lake and Neural Platform (DLNP) is built on a cloud-native, Kubernetes-based architecture that leverages Custom Resource Definitions (CRDs) to manage the entire AI/ML lifecycle. This architecture treats the Kubernetes API as the database layer, with operators fulfilling the desired state defined in CRDs. This approach provides a declarative, scalable, and resilient platform for enterprise AI operations.

## Kubernetes CRD-Based Architecture

### Core Philosophy

The platform adopts a Kubernetes-native approach where:
- **CRDs as Schema**: Custom Resource Definitions define the data model for all platform entities
- **Operators as Business Logic**: Kubernetes operators implement the business logic to reconcile desired state
- **etcd as Database**: Kubernetes' etcd serves as the distributed database for all platform data
- **Controllers as Backend Services**: Controllers watch CRDs and execute necessary actions

### Benefits of CRD-Based Architecture

1. **Declarative Management**: Users declare desired state; the platform ensures it's achieved
2. **Built-in Versioning**: Kubernetes provides native versioning and rollback capabilities
3. **Event-Driven**: Changes trigger events that operators respond to automatically
4. **Self-Healing**: Kubernetes ensures resources remain in desired state
5. **Scalability**: Leverages Kubernetes' proven scalability patterns
6. **Consistency**: Single source of truth through etcd with strong consistency guarantees

## Custom Resource Definitions

### Core Platform CRDs

#### 1. AIModel CRD
```yaml
apiVersion: nstarx.io/v1alpha1
kind: AIModel
metadata:
  name: sentiment-analyzer-v2
  namespace: ml-models
spec:
  type: tensorflow
  version: "2.0"
  framework:
    name: tensorflow
    version: "2.13.0"
  storage:
    location: s3://models/sentiment/v2
    format: savedmodel
  serving:
    replicas: 3
    resources:
      requests:
        cpu: "2"
        memory: "4Gi"
        nvidia.com/gpu: "1"
      limits:
        cpu: "4"
        memory: "8Gi"
        nvidia.com/gpu: "1"
  metrics:
    accuracy: 0.95
    latencyP95: 50ms
  endpoints:
    - name: predict
      path: /v1/models/sentiment/predict
      method: POST
status:
  phase: Serving
  replicas: 3
  endpoints:
    - http://sentiment-analyzer.ml-models.svc.cluster.local:8080
  conditions:
    - type: Ready
      status: "True"
      lastTransitionTime: "2024-01-15T10:00:00Z"
```

#### 2. DataPipeline CRD
```yaml
apiVersion: nstarx.io/v1alpha1
kind: DataPipeline
metadata:
  name: customer-etl-pipeline
  namespace: data-processing
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  source:
    type: database
    config:
      driver: postgresql
      connection: 
        secretRef:
          name: customer-db-credentials
      query: "SELECT * FROM customers WHERE updated_at > :last_run"
  transformations:
    - type: clean
      config:
        removeNulls: true
        deduplication: true
    - type: enrich
      config:
        lookupTable: customer_segments
        joinKey: customer_id
  sink:
    type: datalake
    config:
      path: s3://datalake/customers/processed/
      format: parquet
      partitionBy: 
        - year
        - month
        - day
  resources:
    driver:
      cpu: "2"
      memory: "8Gi"
    executor:
      instances: 4
      cpu: "4"
      memory: "16Gi"
status:
  lastRun: "2024-01-15T02:00:00Z"
  state: Completed
  recordsProcessed: 1523456
  duration: "15m32s"
```

#### 3. MLWorkflow CRD
```yaml
apiVersion: nstarx.io/v1alpha1
kind: MLWorkflow
metadata:
  name: fraud-detection-training
  namespace: ml-workflows
spec:
  type: training
  trigger:
    type: schedule
    schedule: "0 0 * * 0"  # Weekly on Sunday
  stages:
    - name: data-preparation
      type: spark-job
      config:
        mainClass: com.nstarx.ml.DataPrep
        arguments:
          - --input=s3://raw-data/transactions/
          - --output=s3://prepared-data/fraud/
    - name: feature-engineering
      type: python-job
      config:
        image: nstarx/feature-eng:latest
        script: |
          import pandas as pd
          from feature_engine import create_features
          # Feature engineering logic
    - name: model-training
      type: ml-training
      config:
        framework: pytorch
        distributedTraining:
          workers: 4
          backend: nccl
        hyperparameters:
          learning_rate: 0.001
          batch_size: 256
          epochs: 100
    - name: model-evaluation
      type: evaluation
      config:
        metrics:
          - accuracy
          - precision
          - recall
          - f1_score
        threshold: 0.85
    - name: model-registration
      type: model-registry
      config:
        name: fraud-detector
        tags:
          - production-ready
          - v3.0
  resources:
    default:
      cpu: "4"
      memory: "16Gi"
    training:
      nvidia.com/gpu: "4"
      memory: "64Gi"
status:
  phase: Running
  currentStage: model-training
  startTime: "2024-01-14T00:00:00Z"
  stages:
    - name: data-preparation
      status: Completed
      duration: "45m"
    - name: feature-engineering
      status: Completed
      duration: "30m"
    - name: model-training
      status: Running
      progress: 67
```

#### 4. AIZone CRD
```yaml
apiVersion: nstarx.io/v1alpha1
kind: AIZone
metadata:
  name: production-inference-zone
  namespace: ai-zones
spec:
  type: inference
  isolation: namespace
  resources:
    compute:
      nodeSelector:
        nstarx.io/zone: inference
        nvidia.com/gpu.product: NVIDIA-A100-SXM4-80GB
      limits:
        nvidia.com/gpu: "8"
        cpu: "128"
        memory: "512Gi"
    storage:
      class: fast-ssd
      size: "10Ti"
  networking:
    ingress:
      enabled: true
      class: nginx
      tls:
        enabled: true
        secretName: inference-tls
    serviceMesh:
      enabled: true
      mtls: true
  security:
    networkPolicies:
      enabled: true
      allowFrom:
        - namespaceSelector:
            matchLabels:
              nstarx.io/tier: frontend
    podSecurityPolicy:
      enabled: true
      runAsNonRoot: true
      readOnlyRootFilesystem: true
  monitoring:
    prometheus:
      enabled: true
      retention: 30d
    grafana:
      enabled: true
      dashboards:
        - inference-metrics
        - gpu-utilization
  models:
    - name: fraud-detector
      version: "3.0"
      replicas: 5
      autoscaling:
        enabled: true
        minReplicas: 3
        maxReplicas: 10
        metrics:
          - type: cpu
            targetAverageUtilization: 70
          - type: custom
            name: inference_queue_depth
            targetValue: "100"
status:
  phase: Active
  allocatedResources:
    gpu: "6"
    cpu: "96"
    memory: "384Gi"
  models:
    - name: fraud-detector
      ready: 5
      available: 5
      endpoints:
        - https://fraud-detector.inference.nstarx.io
```

#### 5. FeatureStore CRD
```yaml
apiVersion: nstarx.io/v1alpha1
kind: FeatureStore
metadata:
  name: customer-features
  namespace: feature-stores
spec:
  description: "Customer behavioral and demographic features"
  owner: data-science-team
  features:
    - name: customer_lifetime_value
      type: float
      description: "Predicted customer lifetime value"
      source:
        type: batch
        query: |
          SELECT customer_id, 
                 SUM(order_value) * retention_multiplier as ltv
          FROM orders 
          JOIN customer_metrics USING (customer_id)
          GROUP BY customer_id
    - name: purchase_frequency
      type: float
      description: "Average days between purchases"
      source:
        type: streaming
        topic: customer-events
        transformation: |
          def calculate_frequency(events):
            return events.groupby('customer_id').agg(
              lambda x: x['timestamp'].diff().mean()
            )
    - name: churn_probability
      type: float
      description: "Probability of customer churn"
      source:
        type: model
        modelRef:
          name: churn-predictor
          version: "2.1"
  serving:
    online:
      enabled: true
      storage: redis
      ttl: 3600
    offline:
      enabled: true
      storage: parquet
      path: s3://features/customer/
  versioning:
    enabled: true
    retentionDays: 90
  monitoring:
    enabled: true
    metrics:
      - feature_freshness
      - feature_coverage
      - feature_drift
status:
  phase: Serving
  lastUpdate: "2024-01-15T10:30:00Z"
  features:
    total: 3
    serving: 3
    failing: 0
  storage:
    online:
      size: "2.3GB"
      keys: 1523456
    offline:
      size: "156GB"
      partitions: 365
```

## Kubernetes Operators

### Platform Operators Architecture

The platform implements several operators that work together to manage the AI/ML lifecycle:

#### 1. Model Operator
**Responsibilities:**
- Watch AIModel CRDs for changes
- Deploy model serving infrastructure (KServe/Triton)
- Configure auto-scaling based on model specifications
- Monitor model performance and drift
- Handle model versioning and rollback

**Implementation:**
```go
type ModelReconciler struct {
    client.Client
    Scheme *runtime.Scheme
    ModelRegistry ModelRegistryInterface
    ServingPlatform ServingPlatformInterface
}

func (r *ModelReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Fetch the AIModel instance
    model := &nstarxv1alpha1.AIModel{}
    if err := r.Get(ctx, req.NamespacedName, model); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 2. Check if model artifacts exist
    if !r.ModelRegistry.Exists(model.Spec.Storage.Location) {
        return ctrl.Result{}, fmt.Errorf("model artifacts not found")
    }
    
    // 3. Deploy or update serving infrastructure
    serving := r.constructServing(model)
    if err := r.ServingPlatform.Deploy(ctx, serving); err != nil {
        return ctrl.Result{}, err
    }
    
    // 4. Configure monitoring
    if err := r.setupMonitoring(ctx, model); err != nil {
        return ctrl.Result{}, err
    }
    
    // 5. Update status
    model.Status.Phase = "Serving"
    model.Status.Endpoints = serving.GetEndpoints()
    if err := r.Status().Update(ctx, model); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{}, nil
}
```

#### 2. Pipeline Operator
**Responsibilities:**
- Orchestrate data pipeline execution
- Manage Spark/Flink job submissions
- Handle scheduling and triggers
- Monitor pipeline health and performance
- Manage data lineage and quality

#### 3. Workflow Operator
**Responsibilities:**
- Execute ML workflows as directed acyclic graphs (DAGs)
- Coordinate multi-stage processing
- Handle distributed training jobs
- Manage workflow state and checkpoints
- Integrate with experiment tracking

#### 4. Zone Operator
**Responsibilities:**
- Provision isolated AI execution environments
- Configure networking and security policies
- Manage resource quotas and limits
- Deploy monitoring and observability stack
- Handle multi-tenancy requirements

#### 5. Feature Store Operator
**Responsibilities:**
- Synchronize features between online/offline stores
- Execute feature computation jobs
- Monitor feature freshness and quality
- Manage feature versioning
- Handle feature serving infrastructure

## Backend Architecture as Kubernetes Resources

### Data Layer (Kubernetes as Database)

The platform leverages Kubernetes' etcd as the primary data store through CRDs:

```yaml
# All platform data is stored as Kubernetes resources
# Example: User preferences stored as ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-preferences-john-doe
  namespace: user-data
  labels:
    nstarx.io/user-id: "12345"
    nstarx.io/type: "preferences"
data:
  theme: "dark"
  language: "en"
  notifications: "email,slack"
  dashboards: |
    - name: "ML Monitoring"
      widgets: ["model-performance", "data-drift", "resource-usage"]
    - name: "Cost Analysis"
      widgets: ["gpu-costs", "storage-costs", "compute-trends"]
```

### Service Layer (Controllers and Operators)

Services are implemented as Kubernetes controllers that watch resources and execute business logic:

```go
// Example: Workflow execution controller
type WorkflowController struct {
    client.Client
    Scheme *runtime.Scheme
    WorkflowEngine WorkflowEngineInterface
}

func (c *WorkflowController) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&nstarxv1alpha1.MLWorkflow{}).
        Watches(&source.Kind{Type: &batchv1.Job{}}, 
            handler.EnqueueRequestForOwner(mgr.GetScheme(), mgr.GetRESTMapper(), 
                &nstarxv1alpha1.MLWorkflow{}, handler.OnlyControllerOwner())).
        Complete(c)
}
```

### API Layer (Kubernetes API Extension)

The platform extends Kubernetes API to provide a unified interface:

```yaml
# API aggregation for custom endpoints
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1alpha1.nstarx.io
spec:
  service:
    name: nstarx-api
    namespace: nstarx-system
  group: nstarx.io
  version: v1alpha1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 1000
  versionPriority: 15
```

## Security Architecture

### RBAC Integration

The platform leverages Kubernetes RBAC for fine-grained access control:

```yaml
# Role for data scientists
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nstarx:data-scientist
rules:
  - apiGroups: ["nstarx.io"]
    resources: ["aimodels", "mlworkflows", "featurestores"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: ["nstarx.io"]
    resources: ["aizones"]
    verbs: ["get", "list", "use"]
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list"]
    
---
# Role binding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: data-scientists
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nstarx:data-scientist
subjects:
  - kind: Group
    name: data-scientists
    apiGroup: rbac.authorization.k8s.io
```

### Network Policies

Security isolation through Kubernetes network policies:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ai-zone-isolation
  namespace: production-inference
spec:
  podSelector:
    matchLabels:
      nstarx.io/zone: production-inference
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              nstarx.io/tier: frontend
        - podSelector:
            matchLabels:
              nstarx.io/component: gateway
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              nstarx.io/tier: data
      ports:
        - protocol: TCP
          port: 5432
```

## Scalability and Performance

### Horizontal Scaling

The platform scales horizontally through Kubernetes native mechanisms:

```yaml
# HPA for model serving
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fraud-model-hpa
  namespace: model-serving
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fraud-model-server
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: inference_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
        - type: Pods
          value: 10
          periodSeconds: 60
```

### Resource Optimization

Efficient resource utilization through Kubernetes scheduling:

```yaml
# Priority classes for workload scheduling
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: nstarx-production-inference
value: 1000
globalDefault: false
description: "Priority class for production inference workloads"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: nstarx-batch-training
value: 500
globalDefault: false
description: "Priority class for batch training jobs"

---
# Node affinity for GPU workloads
apiVersion: v1
kind: Pod
metadata:
  name: gpu-training-job
spec:
  priorityClassName: nstarx-batch-training
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: nvidia.com/gpu.product
                operator: In
                values:
                  - NVIDIA-A100-SXM4-80GB
                  - NVIDIA-V100-SXM2-32GB
  tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
```

## Monitoring and Observability

### Prometheus Integration

The platform integrates deeply with Prometheus for metrics:

```yaml
# ServiceMonitor for model metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: ai-model-metrics
  namespace: nstarx-system
spec:
  selector:
    matchLabels:
      nstarx.io/component: model-server
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_label_model_name]
          targetLabel: model_name
        - sourceLabels: [__meta_kubernetes_pod_label_model_version]
          targetLabel: model_version
```

### Custom Metrics

Platform-specific metrics exposed through Kubernetes metrics API:

```go
// Custom metrics provider
type MetricsProvider struct {
    client client.Client
}

func (p *MetricsProvider) GetMetricByName(name string, namespace string) (*custom_metrics.MetricValue, error) {
    switch name {
    case "model_inference_latency":
        return p.getModelLatency(namespace)
    case "pipeline_throughput":
        return p.getPipelineThroughput(namespace)
    case "feature_staleness":
        return p.getFeatureStaleness(namespace)
    }
    return nil, fmt.Errorf("metric %s not found", name)
}
```

## Multi-Tenancy

### Namespace Isolation

Each tenant gets isolated namespaces with resource quotas:

```yaml
# Tenant namespace with quotas
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-acme-corp
  labels:
    nstarx.io/tenant: acme-corp
    nstarx.io/tier: enterprise

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: tenant-acme-corp
spec:
  hard:
    requests.cpu: "1000"
    requests.memory: "10Ti"
    requests.nvidia.com/gpu: "20"
    persistentvolumeclaims: "100"
    services: "50"
    nstarx.io/aimodels: "100"
    nstarx.io/mlworkflows: "500"
    nstarx.io/datapipelines: "200"
```

### Hierarchical Resource Management

Resource management through hierarchical namespaces:

```yaml
# HNC configuration for tenant hierarchy
apiVersion: hnc.x-k8s.io/v1alpha2
kind: HierarchicalConfiguration
metadata:
  name: hierarchy
  namespace: tenant-acme-corp
spec:
  parent: nstarx-tenants
  allowCascadingDeletion: false
  labels:
    - key: nstarx.io/tenant
      value: acme-corp
    - key: nstarx.io/billing-id
      value: "acme-12345"
```

## Disaster Recovery

### Backup Strategy

Automated backup of all CRDs and platform state:

```yaml
# Velero backup configuration
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: nstarx-platform-backup
  namespace: velero
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  template:
    includedNamespaces:
      - nstarx-system
      - tenant-*
      - model-serving
      - feature-stores
    includedResources:
      - aimodels.nstarx.io
      - mlworkflows.nstarx.io
      - datapipelines.nstarx.io
      - featurestores.nstarx.io
      - aizones.nstarx.io
      - persistentvolumeclaims
      - configmaps
      - secrets
    storageLocation: nstarx-backup-location
    ttl: 720h  # 30 days retention
```

## Platform Extension Points

### Custom Controllers

Teams can extend the platform by adding custom controllers:

```go
// Example: Custom controller for A/B testing
type ABTestController struct {
    client.Client
    Scheme *runtime.Scheme
}

func (c *ABTestController) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // Custom logic for A/B test management
    abtest := &nstarxv1alpha1.ABTest{}
    if err := c.Get(ctx, req.NamespacedName, abtest); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // Deploy multiple model versions
    for _, variant := range abtest.Spec.Variants {
        if err := c.deployVariant(ctx, abtest, variant); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // Configure traffic splitting
    if err := c.configureTrafficSplit(ctx, abtest); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{RequeueAfter: time.Minute}, nil
}
```

### Webhook Validations

Custom validations through admission webhooks:

```go
// Validation webhook for AI models
type ModelValidator struct {
    Client client.Client
}

func (v *ModelValidator) Handle(ctx context.Context, req admission.Request) admission.Response {
    model := &nstarxv1alpha1.AIModel{}
    if err := json.Unmarshal(req.Object.Raw, model); err != nil {
        return admission.Errored(http.StatusBadRequest, err)
    }
    
    // Validate model specifications
    if model.Spec.Serving.Replicas < 1 {
        return admission.Denied("at least 1 replica required")
    }
    
    // Validate resource requests
    if model.Spec.Serving.Resources.Requests.Memory().IsZero() {
        return admission.Denied("memory requests must be specified")
    }
    
    // Check model artifact existence
    if !v.modelExists(model.Spec.Storage.Location) {
        return admission.Denied("model artifacts not found in specified location")
    }
    
    return admission.Allowed("")
}
```

## Conclusion

The Kubernetes CRD-based architecture provides a robust, scalable, and extensible foundation for the NStarX platform. By treating Kubernetes as both the orchestration layer and the database, the platform achieves:

1. **Declarative Management**: All resources defined as desired state
2. **Built-in Resilience**: Kubernetes ensures resources remain healthy
3. **Extensibility**: Easy to add new resource types and controllers
4. **Unified API**: Single API for all platform operations
5. **Native Integration**: Leverages Kubernetes ecosystem tools
6. **Scalability**: Proven Kubernetes scalability patterns
7. **Security**: Built-in RBAC and network policies
8. **Observability**: Native Prometheus/Grafana integration

This architecture positions the NStarX platform as a truly cloud-native AI/ML platform that can scale from small teams to enterprise deployments while maintaining operational simplicity and reliability.