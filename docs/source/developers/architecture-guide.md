# Developer Architecture Guide

This guide provides detailed technical information for developers working on EchoSkyPilot.

## Module Structure

### Core Module Dependencies

```mermaid
graph TD
    subgraph "sky/"
        CLI[cli.py] --> Core[core.py]
        Core --> Task[task.py] 
        Core --> Resources[resources.py]
        Core --> Backends[backends/]
        Core --> Clouds[clouds/]
        Core --> Execution[execution.py]
        
        Task --> DAG[dag.py]
        Resources --> Catalog[catalog/]
        
        Backends --> BackendUtils[backends/backend_utils.py]
        Backends --> CloudVM[backends/cloud_vm_ray_backend.py]
        Backends --> LocalDocker[backends/local_docker_backend.py]
        
        Clouds --> CloudBase[clouds/cloud.py]
        Clouds --> AWS[clouds/aws.py]
        Clouds --> GCP[clouds/gcp.py]
        Clouds --> Azure[clouds/azure.py]
        Clouds --> K8s[clouds/kubernetes.py]
        
        Jobs[jobs/] --> Controller[jobs/controller.py]
        Jobs --> Scheduler[jobs/scheduler.py]
        Jobs --> Server[jobs/server/]
    end
```

### Key Classes and Interfaces

```mermaid
classDiagram
    class Sky {
        +launch(task: Task, cluster_name: str)
        +exec(task: Task, cluster_name: str)
        +stop(cluster_name: str)
        +status(cluster_names: List[str])
        +down(cluster_name: str)
    }
    
    class Task {
        +resources: Resources
        +setup: List[str]
        +run: List[str]
        +workdir: Optional[str]
        +num_nodes: int
        +docker_image: Optional[str]
        +env_vars: Dict[str, str]
    }
    
    class Resources {
        +cloud: Optional[str]
        +region: Optional[str]
        +zone: Optional[str] 
        +instance_type: Optional[str]
        +accelerators: Optional[str]
        +cpus: Optional[Union[int, float]]
        +memory: Optional[str]
        +disk_size: Optional[int]
        +use_spot: Optional[bool]
        +spot_recovery: Optional[str]
    }
    
    class Backend {
        <<interface>>
        +provision(cluster_name: str, task: Task, dryrun: bool)
        +teardown(cluster_name: str, terminate: bool)  
        +execute(cluster_name: str, task: Task, detach_run: bool)
        +sync_workdir(cluster_name: str, workdir: str)
        +get_cluster_status(cluster_names: List[str])
    }
    
    class Cloud {
        <<interface>>
        +instance_type_exists(instance_type: str) bool
        +get_feasible_launchable_resources(resources: Resources)
        +make_deploy_resources_variables(resources: Resources, cluster_name: str, region: str, zones: List[str])
        +get_image_id(region: str, instance_type: str) str
        +get_default_instance_type() str
    }
    
    Sky --> Task
    Sky --> Backend
    Backend --> Cloud
    Task --> Resources
```

## Backend Implementation Guide

### Creating a New Backend

To add a new backend, implement the `Backend` interface:

```python
from sky.backends import backend

class MyCustomBackend(backend.Backend):
    """Custom backend implementation."""
    
    def __init__(self):
        super().__init__()
        
    def provision(self, cluster_name: str, task: 'Task', dryrun: bool = False):
        """Provision resources for the cluster."""
        # Implementation here
        pass
        
    def teardown(self, cluster_name: str, terminate: bool = False):
        """Tear down the cluster.""" 
        # Implementation here
        pass
        
    def execute(self, cluster_name: str, task: 'Task', detach_run: bool = False):
        """Execute the task on the cluster."""
        # Implementation here  
        pass
        
    def sync_workdir(self, cluster_name: str, workdir: str):
        """Sync local workdir to remote cluster."""
        # Implementation here
        pass
```

### Backend Selection Logic

```mermaid
flowchart TD
    Start[Task Submitted] --> CheckLocal{Local Backend Requested?}
    CheckLocal -->|Yes| LocalBackend[Use Local Docker Backend]
    CheckLocal -->|No| CheckK8s{Kubernetes Resources?}
    CheckK8s -->|Yes| K8sBackend[Use Kubernetes Backend] 
    CheckK8s -->|No| CloudBackend[Use Cloud VM Backend]
    
    LocalBackend --> Validate[Validate Configuration]
    K8sBackend --> Validate
    CloudBackend --> Validate
    
    Validate --> Execute[Execute with Selected Backend]
```

## Cloud Provider Integration

### Adding a New Cloud Provider

1. **Create Cloud Adapter**: Implement the `Cloud` interface in `sky/clouds/`

```python
from sky.clouds import cloud

class MyCloud(cloud.Cloud):
    """My custom cloud provider."""
    
    @classmethod  
    def _cloud_unsupported_features(cls):
        """Return unsupported features for this cloud."""
        return {
            cloud.CloudImplementationFeatures.STOP: 'My cloud does not support stopping instances.',
            cloud.CloudImplementationFeatures.MULTI_NODE: 'Multi-node not yet supported.',
        }
    
    def instance_type_exists(self, instance_type: str) -> bool:
        """Check if instance type exists."""
        # Implementation here
        pass
        
    def get_feasible_launchable_resources(self, resources):
        """Get feasible resources for launching."""
        # Implementation here
        pass
```

2. **Add Provisioning Logic**: Create provisioner in `sky/provision/mycloud/`

3. **Update Registry**: Register the cloud in `sky/clouds/__init__.py`

### Cloud Integration Architecture

```mermaid
graph TB
    subgraph "Cloud Integration"
        CloudRegistry[Cloud Registry]
        CloudFactory[Cloud Factory]
        CloudInterface[Cloud Interface]
    end
    
    subgraph "Provider Implementation"
        MyCloud[MyCloud Adapter]
        Provisioner[MyCloud Provisioner]
        Auth[Authentication]
        Pricing[Pricing Catalog]
    end
    
    subgraph "Infrastructure APIs"
        ComputeAPI[Compute API]
        NetworkAPI[Network API]
        StorageAPI[Storage API]
        IAMAPI[IAM API]
    end
    
    CloudRegistry --> CloudFactory
    CloudFactory --> CloudInterface
    CloudInterface --> MyCloud
    
    MyCloud --> Provisioner
    MyCloud --> Auth
    MyCloud --> Pricing
    
    Provisioner --> ComputeAPI
    Provisioner --> NetworkAPI
    Provisioner --> StorageAPI
    Auth --> IAMAPI
```

## Job Management System

### Managed Jobs Architecture

```mermaid
graph TB
    subgraph "Job Management"
        JobController[Job Controller]
        JobScheduler[Job Scheduler]
        JobQueue[Job Queue] 
        JobDB[(Job State DB)]
    end
    
    subgraph "Job Execution"
        JobRunner[Job Runner]
        SpotController[Spot Controller]
        RecoveryManager[Recovery Manager]
    end
    
    subgraph "Monitoring"
        HealthChecker[Health Checker]
        LogCollector[Log Collector]
        MetricsCollector[Metrics Collector]
    end
    
    JobController --> JobScheduler
    JobScheduler --> JobQueue
    JobQueue --> JobDB
    
    JobScheduler --> JobRunner
    JobRunner --> SpotController
    SpotController --> RecoveryManager
    
    JobRunner --> HealthChecker
    JobRunner --> LogCollector
    JobRunner --> MetricsCollector
    
    RecoveryManager --> JobQueue
```

### Job State Management

```mermaid
stateDiagram-v2
    [*] --> PENDING
    PENDING --> SUBMITTED : Controller picks up job
    SUBMITTED --> STARTING : Resources allocated
    STARTING --> RUNNING : Task execution begins
    RUNNING --> SUCCEEDED : Task completes successfully
    RUNNING --> FAILED : Task fails
    RUNNING --> CANCELLED : User cancellation
    RUNNING --> RECOVERING : Spot preemption detected
    RECOVERING --> RUNNING : Recovery successful  
    RECOVERING --> FAILED : Recovery failed
    STARTING --> FAILED : Setup failure
    SUBMITTED --> FAILED : Resource allocation failure
    
    SUCCEEDED --> [*]
    FAILED --> [*]
    CANCELLED --> [*]
```

## Serving System

### Model Serving Components

```mermaid
graph TB
    subgraph "Serve API"
        ServeAPI[Serve REST API]
        ServiceManager[Service Manager]
        ConfigManager[Config Manager]
    end
    
    subgraph "Load Balancing"
        LoadBalancer[Sky Serve LB]
        RequestRouter[Request Router] 
        HealthChecker[Health Checker]
    end
    
    subgraph "Replica Management"
        ReplicaManager[Replica Manager]
        AutoScaler[Auto Scaler]
        ReplicaMonitor[Replica Monitor]
    end
    
    subgraph "Model Replicas"
        Replica1[Replica 1]
        Replica2[Replica 2]
        ReplicaN[Replica N]
    end
    
    ServeAPI --> ServiceManager
    ServiceManager --> ConfigManager
    ServiceManager --> LoadBalancer
    
    LoadBalancer --> RequestRouter
    LoadBalancer --> HealthChecker
    
    ServiceManager --> ReplicaManager
    ReplicaManager --> AutoScaler
    ReplicaManager --> ReplicaMonitor
    
    ReplicaManager --> Replica1
    ReplicaManager --> Replica2
    ReplicaManager --> ReplicaN
    
    RequestRouter --> Replica1
    RequestRouter --> Replica2
    RequestRouter --> ReplicaN
```

### Auto-scaling Logic

```mermaid
flowchart TD
    Start[Monitor Request Load] --> CollectMetrics[Collect Metrics]
    CollectMetrics --> AnalyzeLoad{Load Analysis}
    AnalyzeLoad -->|High Load| ScaleUp[Scale Up Decision]
    AnalyzeLoad -->|Low Load| ScaleDown[Scale Down Decision]
    AnalyzeLoad -->|Normal Load| Continue[Continue Monitoring]
    
    ScaleUp --> CheckLimits[Check Resource Limits]
    ScaleDown --> CheckMin[Check Min Replicas]
    
    CheckLimits --> CreateReplica[Create New Replica]
    CheckMin --> RemoveReplica[Remove Replica]
    
    CreateReplica --> UpdateLB[Update Load Balancer]
    RemoveReplica --> UpdateLB
    
    UpdateLB --> Continue
    Continue --> CollectMetrics
```

## Data Management

### Storage Integration

```mermaid
graph TB
    subgraph "Storage Abstraction"
        StorageManager[Storage Manager]
        StorageBackend[Storage Backend Interface]
    end
    
    subgraph "Cloud Storage"
        S3[Amazon S3]
        GCS[Google Cloud Storage] 
        AzureBlob[Azure Blob Storage]
        MinIO[MinIO/S3-Compatible]
    end
    
    subgraph "Local Storage"
        LocalFS[Local File System]
        NFS[Network File System]
        HDFS[Hadoop Distributed FS]
    end
    
    subgraph "Data Operations"
        Upload[Upload Data]
        Download[Download Data]
        Sync[Sync Operations]
        Mount[Mount Operations]
    end
    
    StorageManager --> StorageBackend
    StorageBackend --> S3
    StorageBackend --> GCS
    StorageBackend --> AzureBlob
    StorageBackend --> MinIO
    StorageBackend --> LocalFS
    StorageBackend --> NFS
    StorageBackend --> HDFS
    
    StorageManager --> Upload
    StorageManager --> Download
    StorageManager --> Sync
    StorageManager --> Mount
```

## Configuration Management

### Configuration Hierarchy

```mermaid
flowchart TD
    subgraph "Configuration Sources"
        DefaultConfig[Default Configuration]
        GlobalConfig[Global Config File]
        UserConfig[User Config File]
        EnvVars[Environment Variables]
        CLIArgs[CLI Arguments]
    end
    
    subgraph "Configuration Processing" 
        ConfigLoader[Configuration Loader]
        ConfigMerger[Configuration Merger]
        ConfigValidator[Configuration Validator]
    end
    
    subgraph "Runtime Configuration"
        RuntimeConfig[Runtime Configuration]
        CloudConfig[Cloud Configuration]
        BackendConfig[Backend Configuration]
    end
    
    DefaultConfig --> ConfigLoader
    GlobalConfig --> ConfigLoader
    UserConfig --> ConfigLoader
    EnvVars --> ConfigLoader
    CLIArgs --> ConfigLoader
    
    ConfigLoader --> ConfigMerger
    ConfigMerger --> ConfigValidator
    ConfigValidator --> RuntimeConfig
    
    RuntimeConfig --> CloudConfig
    RuntimeConfig --> BackendConfig
```

## Testing Architecture

### Test Structure

```mermaid
graph TB
    subgraph "Unit Tests"
        CoreTests[Core Module Tests]
        BackendTests[Backend Tests]
        CloudTests[Cloud Provider Tests]
        UtilTests[Utility Tests]
    end
    
    subgraph "Integration Tests"
        E2ETests[End-to-End Tests]
        CloudIntegration[Cloud Integration Tests]
        BackendIntegration[Backend Integration Tests]
    end
    
    subgraph "Performance Tests"
        LoadTests[Load Tests]
        ScaleTests[Scale Tests]
        LatencyTests[Latency Tests]
    end
    
    subgraph "Test Infrastructure"
        Fixtures[Test Fixtures]
        Mocks[Mock Services]
        TestData[Test Data]
    end
    
    CoreTests --> Fixtures
    BackendTests --> Mocks
    CloudTests --> TestData
    
    E2ETests --> CloudIntegration
    CloudIntegration --> BackendIntegration
    
    LoadTests --> ScaleTests
    ScaleTests --> LatencyTests
```

## Development Workflow

### Code Contribution Flow

```mermaid
gitgraph
    commit id: "main"
    branch feature
    checkout feature
    commit id: "Start feature"
    commit id: "Implement changes"
    commit id: "Add tests"
    commit id: "Update docs"
    checkout main
    commit id: "Other changes"
    checkout feature
    merge main
    commit id: "Fix conflicts"
    checkout main
    merge feature
    commit id: "Merge feature"
```

### Build and Release Pipeline

```mermaid
flowchart TD
    Start[Code Commit] --> Lint[Code Linting]
    Lint --> UnitTest[Unit Tests]
    UnitTest --> IntegrationTest[Integration Tests]
    IntegrationTest --> Build[Build Package]
    Build --> SecurityScan[Security Scan]
    SecurityScan --> Deploy{Deploy?}
    Deploy -->|Staging| StagingDeploy[Deploy to Staging]
    Deploy -->|Production| ProdDeploy[Deploy to Production]
    StagingDeploy --> StagingTest[Staging Tests]
    StagingTest --> ProdDeploy
    ProdDeploy --> Monitor[Monitor Release]
```

This architecture guide provides the foundation for understanding and contributing to EchoSkyPilot's codebase. For specific implementation details, refer to the inline code documentation and examples in the repository.