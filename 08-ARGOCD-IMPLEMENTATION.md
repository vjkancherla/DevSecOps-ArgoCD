# Current ArgoCD Setup Documentation

## Overview
This document captures the current state of the ArgoCD setup and application configuration before implementing enhanced versioning and feature branch support.

## Current ArgoCD Application Configuration

### File: `application.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: petclinic
  namespace: argo-cd
spec:
  project: default
  source:
    repoURL: registry-1.docker.io/vjkancherla
    chart: petclinic
    # see - https://argo-cd.readthedocs.io/en/stable/user-guide/tracking_strategies/#helm
    targetRevision: "0.1.*"
    # No helm values needed - chart already has correct image
  destination:
    server: https://kubernetes.default.svc
    namespace: petclinic-dev
  syncPolicy:
    automated: {}
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

## Current Architecture Flow

### 1. CI/CD Pipeline (Jenkins)
- **Trigger**: Feature branch is pushed to origin
- **Process**: Jenkins Multi-Branch pipeline is triggered manually
- **DevSecOps Tasks**:
  - Code checkout
  - Maven compile, test, and package
  - SonarQube code analysis
  - GitSecrets: scan for hardcoded secrets
  - OWASP code analysis
  - Build image with Kaniko
  - Scan image with Trivy
  - Publish image to Docker Hub
  - Scan Helm chart with Trivy
  - Helm chart dry-run
  - Package, tag, and publish Helm chart to Docker Hub

### 2. Current Versioning Strategy
- **Chart Versions**: Uses pattern `0.1.*` 
- **Image Tags**: Based on Git commit hash: `${GIT_COMMIT_HASH_SHORT}`
- **Chart Publishing**: Charts are published to Docker Hub OCI registry
- **Repository**: `registry-1.docker.io/vjkancherla/petclinic`

### 3. ArgoCD Deployment
- **Monitoring**: ArgoCD watches Docker Hub helm repo for updated versions
- **Target Revision**: `"0.1.*"` - picks up any chart version matching this pattern
- **Deployment**: ArgoCD automatically deploys updated helm charts
- **Namespace**: Single namespace `petclinic-dev`
- **Sync Policy**: Basic automated sync enabled

## Current Limitations

### 1. Single Environment
- **Problem**: All deployments go to the same namespace (`petclinic-dev`)
- **Impact**: No isolation between different feature branches or developers
- **Conflict Risk**: Multiple developers working on different features deploy to same environment

### 2. Version Strategy Issues
- **Problem**: `"0.1.*"` pattern is too broad
- **Impact**: ArgoCD picks up charts from any branch that publishes to `0.1.x` range
- **Unpredictability**: Cannot control which feature's chart gets deployed

### 3. No Branch Isolation
- **Problem**: No facility for per-feature-branch deployments
- **Impact**: Developers cannot test their features in isolation
- **Testing Limitation**: Cannot run parallel feature development

### 4. Missing Auto-Healing and Pruning
- **Problem**: Basic sync policy without advanced options
- **Missing Features**: 
  - No auto-pruning of removed resources
  - No auto-healing when cluster state drifts
  - No retry logic for failed syncs

### 5. No Git Traceability
- **Problem**: Chart versions don't reference Git commits
- **Impact**: Difficult to trace which code is running in which environment
- **Rollback Difficulty**: Hard to identify which commit to rollback to

## Current Jenkins Pipeline Versioning

### From `Jenkinsfile` Analysis:
```groovy
// Current versioning logic
CHART_VERSION = sh(script: """
  if git describe --tags --exact-match HEAD 2>/dev/null; then
    git describe --tags --exact-match HEAD | sed 's/^v//'
  else 
    echo "0.1.${BUILD_NUMBER}+git.${GIT_COMMIT_HASH_SHORT}"
  fi
""", returnStdout: true).trim()
```

### Current Behavior:
- **Tagged Releases**: Use git tag as version
- **Non-Tagged Commits**: Use format `0.1.{BUILD_NUMBER}+git.{COMMIT_HASH}`
- **Image Tag**: `${GIT_COMMIT_HASH_SHORT}`
- **Chart Updates**: Jenkins updates `values.yaml` with correct image tag before publishing

## Current Docker Hub Chart Storage

### Registry Details:
- **Registry**: `registry-1.docker.io`
- **Organization**: `vjkancherla`
- **Chart Name**: `petclinic`
- **Chart Format**: OCI registry format
- **Chart Location**: `oci://registry-1.docker.io/vjkancherla/petclinic`

### Chart Publishing Process:
1. Jenkins packages chart with version and app version
2. Updates `values.yaml` with specific image tag
3. Publishes to Docker Hub OCI registry
4. ArgoCD monitors this registry for new versions

## Current Sync Behavior

### ArgoCD Monitoring:
- **Watch Target**: Docker Hub OCI registry
- **Version Pattern**: `"0.1.*"`
- **Sync Trigger**: New chart version appears matching pattern
- **Deployment Target**: `petclinic-dev` namespace
- **Update Strategy**: Automated deployment of latest matching version

### Sync Policy Analysis:
```yaml
syncPolicy:
  automated: {}  # Basic automation enabled
  syncOptions:
    - CreateNamespace=true     # Auto-create namespace if missing
    - ServerSideApply=true     # Use server-side apply for resources
```

**Missing Advanced Options:**
- No `prune: true` (resources aren't cleaned up when removed from chart)
- No `selfHeal: true` (manual changes to cluster aren't auto-corrected)
- No retry logic for failed syncs
- No propagation policies for resource cleanup

## What Works Well Currently

### ✅ Strengths:
1. **Automated CI/CD**: Complete pipeline from code to deployment
2. **Security Scanning**: Comprehensive security checks (SonarQube, OWASP, Trivy, GitLeaks)
3. **Chart Self-Containment**: Charts include correct image tags
4. **OCI Registry**: Modern chart storage using Docker Hub OCI
5. **Basic GitOps**: ArgoCD automatically deploys new chart versions

## What Needs Improvement

### ❌ Areas for Enhancement:
1. **Environment Isolation**: Need per-branch namespaces
2. **Version Strategy**: Need branch-aware semantic versioning
3. **Git Traceability**: Need commit references in versions
4. **Concurrent Development**: Need support for multiple developers
5. **Advanced Sync Policies**: Need auto-pruning and self-healing
6. **Cleanup Automation**: Need automatic environment cleanup

This documentation provides the baseline for implementing the enhanced ArgoCD setup with feature branch support and improved versioning strategy.