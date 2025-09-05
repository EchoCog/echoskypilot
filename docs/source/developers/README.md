# Architecture Documentation Index

This directory contains comprehensive technical architecture documentation for EchoSkyPilot.

## Documents

### [Technical Architecture](../architecture.md)
Complete system architecture with interactive Mermaid diagrams covering:
- System overview and design principles
- High-level architecture with all components
- Core engine and module structures  
- Cloud provider integration patterns
- Job lifecycle management
- Data flow and synchronization
- Backend architectures
- Model serving pipelines
- API design and security

### [Developer Architecture Guide](architecture-guide.md)
In-depth guide for developers and contributors:
- Module structure and dependencies
- Implementation patterns for extensions
- Backend and cloud provider integration
- Job management system internals
- Testing architecture
- Development workflows

### [Architecture Overview](../architecture-overview.md)
Quick start guide with examples and navigation to full documentation.

## Quick Reference

The architecture follows these key patterns:

1. **Modular Design**: Clear separation between core, backends, and cloud adapters
2. **Plugin Architecture**: Extensible system for new backends and cloud providers
3. **Fault Tolerance**: Built-in recovery and preemption handling
4. **Multi-Cloud**: Unified interface across 16+ cloud providers
5. **Developer Experience**: Simple YAML-based task definitions

## Getting Started

For new developers:
1. Start with [Architecture Overview](../architecture-overview.md) for a quick introduction
2. Review [Technical Architecture](../architecture.md) for comprehensive system understanding  
3. Use [Developer Architecture Guide](architecture-guide.md) for implementation details

For system administrators:
1. Focus on [Cloud Provider Integration](../architecture.md#cloud-provider-integration)
2. Review [Security Architecture](../architecture.md#security-architecture)
3. Understand [Job Lifecycle Management](../architecture.md#job-lifecycle-management)