# AI/ML Deployment Platform Architecture

## Overview

This document describes the architecture of the AI/ML Deployment Platform, which provides comprehensive infrastructure provisioning and management for AI/ML workloads on Kubernetes.

## System Components

### Core Services

1. **AI Converge Builder**
   - Manages AI model deployment pods
   - Handles build status tracking
   - Provides pod lifecycle management

2. **Deployment Manager**
   - Creates and manages Kubernetes deployments
   - Handles zone management
   - Supports deployment updates and redeployments

3. **RAG Manager**
   - Deploys and manages RAG (Retrieval-Augmented Generation) models
   - Provides model discovery and listing

4. **Chat Manager**
   - Processes chat requests
   - Manages conversation sessions
   - Integrates with deployed models

5. **MLflow Manager**
   - Tracks experiment metrics
   - Provides experiment data retrieval

6. **Cost Estimator**
   - Calculates deployment costs
   - Provides cost optimization recommendations

7. **SSE Manager**
   - Handles Server-Sent Events streaming
   - Manages trust dashboard data

### Infrastructure Components

- **Kubernetes**: Container orchestration platform
- **MinIO**: Object storage for model artifacts
- **MLflow**: Experiment tracking and model registry
- **Custom Resources**: CRDs for RAG models and other custom resources

## Architecture Diagrams

View the interactive architecture diagrams in the [Architecture Viewer](./index.html).

## API Endpoints

The platform exposes the following key endpoints:

- `/ai-converge/build`: Create AI model deployment pods
- `/ai-converge/builds/{build_status}`: Get pods by build status
- `/deployments`: Create and manage deployments
- `/deployments/zones`: List available deployment zones
- `/rag/models`: Deploy and manage RAG models
- `/chat`: Process chat requests
- `/mlflow/experiments`: Access MLflow experiment data
- `/cost/estimate`: Get cost estimations
- `/sse/stream`: Server-sent events streaming

## Security Considerations

- CORS middleware enabled for cross-origin requests
- Health and readiness endpoints for monitoring
- Session management for chat conversations
- Secure handling of model artifacts

## Deployment Architecture

The platform supports multiple deployment patterns:

1. **Single Zone Deployment**: Deploy to a specific zone
2. **Multi-Zone Deployment**: Distribute across multiple zones
3. **Auto-Scaling**: Dynamic scaling based on load

## Data Flow

1. User requests come through the FastAPI application
2. Requests are routed to appropriate managers
3. Managers interact with Kubernetes API and custom resources
4. Results are returned through the API response

## Monitoring and Observability

- Health check endpoint: `/health`
- Readiness endpoint: `/readiness`
- SSE streaming for real-time updates
- MLflow integration for model metrics