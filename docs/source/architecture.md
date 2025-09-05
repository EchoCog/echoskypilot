# EchoSkyPilot Technical Architecture

This document provides comprehensive technical architecture documentation for EchoSkyPilot, a framework for running AI workloads on any cloud infrastructure through a unified interface.

## Table of Contents

- [System Overview](#system-overview)
- [High-Level Architecture](#high-level-architecture)  
- [Core Components](#core-components)
- [Cloud Provider Integration](#cloud-provider-integration)
- [Job Lifecycle Management](#job-lifecycle-management)
- [Data Flow Architecture](#data-flow-architecture)
- [Backend Architecture](#backend-architecture)
- [Serving Architecture](#serving-architecture)
- [API Architecture](#api-architecture)
- [Security Architecture](#security-architecture)

## System Overview

EchoSkyPilot is a cloud-agnostic orchestration platform that abstracts away the complexities of running AI/ML workloads across multiple cloud providers and infrastructure types. It provides a unified interface for resource management, job scheduling, and workload execution.

### Key Design Principles

- **Cloud Agnostic**: Single interface for multiple cloud providers
- **Cost Optimization**: Intelligent resource selection and spot instance management  
- **Fault Tolerance**: Auto-recovery and preemption handling
- **Scalability**: Multi-node and distributed execution support
- **Developer Experience**: Simple YAML-based task definitions

## High-Level Architecture

```mermaid
graph TB
    subgraph "User Interface Layer"
        CLI[Sky CLI]
        API[REST API]
        Dashboard[Web Dashboard]
        Python[Python SDK]
    end
    
    subgraph "Core System"
        Core[Core Engine]
        Optimizer[Resource Optimizer]
        Scheduler[Job Scheduler] 
        Controller[Job Controller]
    end
    
    subgraph "Backend Layer"
        CloudVM[Cloud VM Backend]
        K8s[Kubernetes Backend] 
        Docker[Local Docker Backend]
    end
    
    subgraph "Cloud Providers"
        AWS[Amazon Web Services]
        GCP[Google Cloud Platform]
        Azure[Microsoft Azure]
        Others[Other Clouds...]
    end
    
    subgraph "Infrastructure Services"
        Storage[Cloud Storage]
        Registry[Container Registry]
        Monitoring[Monitoring & Logs]
    end
    
    CLI --> Core
    API --> Core
    Dashboard --> Core
    Python --> Core
    
    Core --> Optimizer
    Core --> Scheduler
    Core --> Controller
    
    Optimizer --> CloudVM
    Scheduler --> K8s
    Controller --> Docker
    
    CloudVM --> AWS
    CloudVM --> GCP
    CloudVM --> Azure
    K8s --> AWS
    K8s --> GCP
    K8s --> Azure
    
    AWS --> Storage
    GCP --> Storage
    Azure --> Storage
    Storage --> Registry
    Registry --> Monitoring
```

## Core Components

### Core Engine Architecture

```mermaid
graph TB
    subgraph "Sky Core Module"
        Launch[launch()]
        Exec[exec()]
        Stop[stop()]
        Status[status()]
        Down[down()]
    end
    
    subgraph "Task Management"
        Task[Task Definition]
        DAG[DAG Construction]
        Resources[Resource Requirements]
        Validation[Task Validation]
    end
    
    subgraph "Resource Management"
        Catalog[Cloud Catalog]
        Pricing[Pricing Engine]
        Availability[Availability Checker]
    end
    
    Launch --> Task
    Task --> DAG
    DAG --> Resources
    Resources --> Validation
    
    Launch --> Catalog
    Catalog --> Pricing
    Pricing --> Availability
    
    Exec --> Status
    Stop --> Down
```

### Component Relationships

```mermaid
classDiagram
    class SkyCore {
        +launch(task, cluster_name)
        +exec(task, cluster_name)
        +stop(cluster_name)
        +status(cluster_names)
        +down(cluster_name)
    }
    
    class Task {
        +resources: Resources
        +setup: List[str]
        +run: List[str]
        +workdir: str
        +num_nodes: int
    }
    
    class Resources {
        +cloud: str
        +region: str
        +zone: str
        +instance_type: str
        +accelerators: str
        +cpus: float
        +memory: str
        +disk_size: int
    }
    
    class Backend {
        <<interface>>
        +provision()
        +teardown()
        +execute()
        +sync_workdir()
    }
    
    class CloudVMBackend {
        +provision()
        +teardown()
        +execute()
        +sync_workdir()
    }
    
    class KubernetesBackend {
        +provision()
        +teardown()
        +execute()
        +sync_workdir()
    }
    
    SkyCore --> Task
    Task --> Resources
    SkyCore --> Backend
    Backend <|-- CloudVMBackend
    Backend <|-- KubernetesBackend
```

## Cloud Provider Integration

### Cloud Adapter Architecture

```mermaid
graph TB
    subgraph "Cloud Abstraction Layer"
        CloudInterface[Cloud Interface]
        Provisioner[Provisioner]
        Authenticator[Authentication]
    end
    
    subgraph "AWS Integration"
        AWSCloud[AWS Cloud Adapter]
        EC2[EC2 Provisioner]
        IAM[IAM Auth]
        S3[S3 Storage]
    end
    
    subgraph "GCP Integration"  
        GCPCloud[GCP Cloud Adapter]
        Compute[Compute Engine]
        GCPAuth[Service Account Auth]
        GCS[Cloud Storage]
    end
    
    subgraph "Azure Integration"
        AzureCloud[Azure Cloud Adapter] 
        VM[Virtual Machines]
        AzureAuth[Azure AD Auth]
        Blob[Blob Storage]
    end
    
    subgraph "Kubernetes Integration"
        K8sCloud[Kubernetes Adapter]
        Pods[Pod Management]
        K8sAuth[RBAC Auth]
        PVC[Persistent Volumes]
    end
    
    CloudInterface --> AWSCloud
    CloudInterface --> GCPCloud
    CloudInterface --> AzureCloud
    CloudInterface --> K8sCloud
    
    Provisioner --> EC2
    Provisioner --> Compute
    Provisioner --> VM
    Provisioner --> Pods
    
    Authenticator --> IAM
    Authenticator --> GCPAuth
    Authenticator --> AzureAuth
    Authenticator --> K8sAuth
```

### Multi-Cloud Resource Selection

```mermaid
flowchart TD
    Start([Task Submitted]) --> Parse[Parse Resource Requirements]
    Parse --> Catalog[Query Cloud Catalog]
    Catalog --> Filter[Filter Available Options]
    Filter --> Price[Calculate Pricing]
    Price --> Policy[Apply Admin Policies]
    Policy --> Optimize[Optimize Selection]
    Optimize --> Provision[Provision Resources]
    Provision --> Success{Provisioning Successful?}
    Success -->|Yes| Execute[Execute Task]
    Success -->|No| Retry[Retry with Fallback]
    Retry --> Filter
    Execute --> Monitor[Monitor Execution]
    Monitor --> Cleanup[Cleanup Resources]
    Cleanup --> End([Complete])
```

## Job Lifecycle Management

### Job State Machine

```mermaid
stateDiagram-v2
    [*] --> INIT
    INIT --> PENDING : Submit Job
    PENDING --> SETTING_UP : Resources Allocated
    SETTING_UP --> RUNNING : Environment Ready
    RUNNING --> SUCCEEDED : Job Completes
    RUNNING --> FAILED : Job Fails
    RUNNING --> CANCELLED : User Cancels
    SETTING_UP --> FAILED : Setup Fails
    PENDING --> CANCELLED : User Cancels
    FAILED --> [*]
    SUCCEEDED --> [*]
    CANCELLED --> [*]
    
    RUNNING --> RECOVERING : Spot Preemption
    RECOVERING --> RUNNING : Recovery Successful
    RECOVERING --> FAILED : Recovery Failed
```

### Managed Jobs Architecture

```mermaid
graph TB
    subgraph "Job Management System"
        JobController[Job Controller]
        JobScheduler[Job Scheduler] 
        JobQueue[Job Queue]
        JobDB[(Job Database)]
    end
    
    subgraph "Recovery System"
        SpotHandler[Spot Preemption Handler]
        RecoveryStrategy[Recovery Strategy]
        StateManager[State Manager]
    end
    
    subgraph "Execution Environment"
        Cluster[Sky Cluster]
        Workdir[Work Directory]
        Logs[Log Collection]
    end
    
    JobController --> JobScheduler
    JobScheduler --> JobQueue
    JobQueue --> JobDB
    
    JobController --> SpotHandler
    SpotHandler --> RecoveryStrategy
    RecoveryStrategy --> StateManager
    
    JobScheduler --> Cluster
    Cluster --> Workdir
    Cluster --> Logs
    
    StateManager --> JobDB
```

## Data Flow Architecture

### Task Execution Flow

```mermaid
sequenceDiagram
    participant User
    participant CLI
    participant Core
    participant Backend
    participant Cloud
    participant Cluster
    
    User->>CLI: sky launch task.yaml
    CLI->>Core: parse_and_launch()
    Core->>Core: validate_task()
    Core->>Backend: provision_cluster()
    Backend->>Cloud: create_instances()
    Cloud-->>Backend: instances_ready
    Backend-->>Core: cluster_provisioned
    Core->>Backend: sync_workdir()
    Backend->>Cluster: rsync files
    Core->>Backend: setup_environment()
    Backend->>Cluster: run setup commands
    Core->>Backend: execute_task()
    Backend->>Cluster: run task commands
    Cluster-->>Backend: execution_complete
    Backend-->>Core: task_finished
    Core-->>CLI: success
    CLI-->>User: Task completed
```

### Data Synchronization Flow

```mermaid
flowchart LR
    subgraph Local[Local Environment]
        LocalCode[Source Code]
        LocalData[Data Files]
        LocalConfig[Configuration]
    end
    
    subgraph Transfer[Data Transfer]
        Rsync[Rsync/SCP]
        CloudSync[Cloud Storage Sync]
        GitSync[Git Repository]
    end
    
    subgraph Remote[Remote Cluster]
        WorkDir[~/sky_workdir]
        DataMount[Data Mounts]
        Results[Output Results]
    end
    
    LocalCode --> Rsync
    LocalData --> CloudSync
    LocalConfig --> GitSync
    
    Rsync --> WorkDir
    CloudSync --> DataMount
    GitSync --> WorkDir
    
    WorkDir --> Results
    Results --> LocalData
```

## Backend Architecture

### Backend Interface Design

```mermaid
classDiagram
    class Backend {
        <<abstract>>
        +provision(cluster_name, task, dryrun)
        +teardown(cluster_name, terminate)
        +execute(cluster_name, task, detach_run)
        +sync_workdir(cluster_name, workdir)
        +get_cluster_status(cluster_names)
    }
    
    class CloudVMRayBackend {
        +provision() "Provisions cloud VMs"
        +teardown() "Terminates cloud resources"  
        +execute() "Runs Ray-based execution"
        +sync_workdir() "Syncs via rsync"
        +ray_up() "Ray cluster setup"
        +ray_down() "Ray cluster teardown"
    }
    
    class KubernetesBackend {
        +provision() "Creates K8s resources"
        +teardown() "Deletes K8s resources"
        +execute() "Runs as K8s jobs/pods"
        +sync_workdir() "Mounts or copies files"
        +create_namespace() "Namespace management"
        +apply_manifests() "K8s resource creation"
    }
    
    class LocalDockerBackend {
        +provision() "Pulls Docker images"
        +teardown() "Stops containers"
        +execute() "Runs Docker containers"
        +sync_workdir() "Volume mounts"
        +build_image() "Custom image building"
    }
    
    Backend <|-- CloudVMRayBackend
    Backend <|-- KubernetesBackend  
    Backend <|-- LocalDockerBackend
```

### Execution Environment Setup

```mermaid
flowchart TD
    Start[Backend.provision()] --> CheckCloud[Check Cloud Availability]
    CheckCloud --> CreateResources[Create Cloud Resources]
    CreateResources --> WaitReady[Wait for Resources Ready]
    WaitReady --> InstallDeps[Install Dependencies]
    InstallDeps --> SetupRay{Ray Backend?}
    SetupRay -->|Yes| RayInit[Initialize Ray Cluster]
    SetupRay -->|No| K8sSetup{Kubernetes Backend?}
    K8sSetup -->|Yes| PodSetup[Setup Pod/Job]
    K8sSetup -->|No| DockerSetup[Setup Docker Container]
    RayInit --> SyncFiles[Sync Workdir]
    PodSetup --> SyncFiles
    DockerSetup --> SyncFiles
    SyncFiles --> Ready[Environment Ready]
```

## Serving Architecture

### Model Serving Pipeline

```mermaid
graph TB
    subgraph "Sky Serve System"
        ServeController[Serve Controller]
        LoadBalancer[Load Balancer]
        ServiceRegistry[Service Registry]
    end
    
    subgraph "Replica Management"
        ReplicaManager[Replica Manager]
        AutoScaler[Auto Scaler]
        HealthChecker[Health Checker]
    end
    
    subgraph "Model Instances"
        Replica1[Model Replica 1]
        Replica2[Model Replica 2]
        ReplicaN[Model Replica N]
    end
    
    subgraph "Infrastructure"
        CloudInfra[Cloud Infrastructure]
        K8sCluster[K8s Cluster]
        Monitoring[Metrics & Monitoring]
    end
    
    ServeController --> ReplicaManager
    ReplicaManager --> Replica1
    ReplicaManager --> Replica2
    ReplicaManager --> ReplicaN
    
    LoadBalancer --> Replica1
    LoadBalancer --> Replica2  
    LoadBalancer --> ReplicaN
    
    AutoScaler --> ReplicaManager
    HealthChecker --> ServiceRegistry
    
    Replica1 --> CloudInfra
    Replica2 --> K8sCluster
    ReplicaN --> CloudInfra
    
    Monitoring --> AutoScaler
```

### Request Flow

```mermaid
sequenceDiagram
    participant Client
    participant LB as Load Balancer
    participant Replica as Model Replica
    participant Controller as Serve Controller
    participant Scaler as Auto Scaler
    
    Client->>LB: HTTP Request
    LB->>Replica: Route Request
    Replica->>Replica: Process Request
    Replica-->>LB: Response
    LB-->>Client: HTTP Response
    
    Note over Replica: Monitor Metrics
    Replica->>Controller: Health Status
    Controller->>Scaler: Load Metrics
    Scaler->>Controller: Scale Decision
    Controller->>Replica: Scale Up/Down
```

## API Architecture

### REST API Design

```mermaid
graph TB
    subgraph "API Gateway"
        Router[FastAPI Router]
        Auth[Authentication]
        RateLimit[Rate Limiting]
    end
    
    subgraph "Core Services"
        ClusterAPI[Cluster Management API]
        JobAPI[Job Management API]
        ServeAPI[Serving API]
        StorageAPI[Storage API]
    end
    
    subgraph "Backend Services"
        ClusterService[Cluster Service]
        JobService[Job Service]
        ServeService[Serve Service]
        StorageService[Storage Service]
    end
    
    subgraph "Data Layer"
        StateDB[(State Database)]
        JobDB[(Job Database)]
        MetricsDB[(Metrics Database)]
    end
    
    Router --> Auth
    Auth --> RateLimit
    RateLimit --> ClusterAPI
    RateLimit --> JobAPI
    RateLimit --> ServeAPI
    RateLimit --> StorageAPI
    
    ClusterAPI --> ClusterService
    JobAPI --> JobService
    ServeAPI --> ServeService
    StorageAPI --> StorageService
    
    ClusterService --> StateDB
    JobService --> JobDB
    ServeService --> StateDB
    StorageService --> MetricsDB
```

### API Endpoints Structure

```mermaid
mindmap
  root((Sky API))
    Clusters
      GET /clusters
      GET /clusters/{name}
      POST /clusters
      DELETE /clusters/{name}
      POST /clusters/{name}/exec
    Jobs
      GET /jobs  
      GET /jobs/{id}
      POST /jobs
      DELETE /jobs/{id}
      POST /jobs/{id}/cancel
    Serve
      GET /serve/services
      POST /serve/services
      DELETE /serve/services/{name}
      GET /serve/services/{name}/replicas
    Storage
      GET /storage
      POST /storage/upload
      GET /storage/{path}
      DELETE /storage/{path}
```

## Security Architecture

### Authentication & Authorization Flow

```mermaid
sequenceDiagram
    participant User
    participant CLI
    participant API
    participant Auth as Auth Service
    participant Cloud
    
    User->>CLI: sky launch --cloud aws
    CLI->>API: Request with credentials
    API->>Auth: Validate credentials
    Auth->>Auth: Check permissions
    Auth-->>API: Auth token
    API->>Cloud: Provision with cloud creds
    Cloud-->>API: Resources created
    API-->>CLI: Success
    CLI-->>User: Cluster ready
```

### Multi-Cloud Credential Management

```mermaid
flowchart TD
    subgraph Credentials[Credential Management]
        LocalCreds[Local Credentials]
        CloudCreds[Cloud Provider Creds]
        K8sCreds[Kubernetes Credentials]
    end
    
    subgraph Storage[Secure Storage]
        CredStore[Credential Store]
        Encryption[Encryption at Rest]
        KeyVault[Key Management]
    end
    
    subgraph Access[Access Control]
        RBAC[Role-Based Access]
        Policies[Admin Policies] 
        Audit[Audit Logging]
    end
    
    LocalCreds --> CredStore
    CloudCreds --> CredStore
    K8sCreds --> CredStore
    
    CredStore --> Encryption
    Encryption --> KeyVault
    
    CredStore --> RBAC
    RBAC --> Policies
    Policies --> Audit
```

## Summary

EchoSkyPilot's architecture is designed around several key principles:

1. **Modularity**: Clear separation between core logic, backends, and cloud adapters
2. **Extensibility**: Plugin architecture for adding new cloud providers and backends
3. **Resilience**: Built-in fault tolerance and recovery mechanisms
4. **Performance**: Optimized resource selection and efficient data transfer
5. **Security**: Comprehensive credential management and access controls

The system successfully abstracts the complexity of multi-cloud deployment while providing powerful features for AI workload orchestration, cost optimization, and operational simplicity.