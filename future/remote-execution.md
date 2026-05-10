# Remote Workflow Execution

## Overview

Enable sir.sh to execute workflows on remote machines, containers, or cloud environments, extending its capabilities beyond local execution.

## Use Cases

### 1. CI/CD Integration
Run sir workflows as part of CI/CD pipelines on remote build agents.

### 2. Distributed Testing
Execute test suites across multiple remote machines in parallel.

### 3. Remote Deployment
Deploy applications to remote servers using sir workflows.

### 4. Cloud Execution
Run resource-intensive workflows on cloud VMs or containers.

## Proposed Implementation

### Remote Execution Modes

#### SSH Execution
```bash
sir run deploy --remote ssh://user@server.com
```

#### Docker Execution
```bash
sir run build --remote docker://ubuntu:latest
```

#### Kubernetes Execution
```bash
sir run test --remote k8s://cluster/namespace
```

#### Cloud Functions
```bash
sir run process --remote aws-lambda://function-name
```

### Workflow Configuration

```yaml
description: "Deployable workflow"

# Define remote execution requirements
remote:
  image: "php:8.2-cli"
  requires:
    - git
    - composer
  environment:
    NODE_ENV: production

inputs:
  target:
    type: string
    required: true

steps:
  - task: deploy.app
    with:
      target: "{{target}}"
```

### Remote Command

```bash
# Execute on remote host
sir run workflow --remote user@host

# Execute in Docker container
sir run workflow --remote docker://image

# Execute on Kubernetes
sir run workflow --remote k8s://pod

# Execute with specific configuration
sir run workflow --remote-config ./remote.yml
```

## Architecture

### Remote Executor

```php
interface RemoteExecutor
{
    public function execute(Workflow $workflow, array $inputs): Result;
    public function isAvailable(): bool;
    public function getCapabilities(): array;
}

class SSHExecutor implements RemoteExecutor
{
    public function execute(Workflow $workflow, array $inputs): Result
    {
        // 1. Package workflow and dependencies
        $package = $this->packageWorkflow($workflow);
        
        // 2. Transfer to remote host
        $this->transfer($package);
        
        // 3. Execute remotely
        $result = $this->executeRemote();
        
        // 4. Fetch results
        return $this->fetchResults();
    }
}
```

### Workflow Packaging

Package workflow with all dependencies:

```php
class WorkflowPackager
{
    public function package(Workflow $workflow): Package
    {
        return new Package([
            'workflow' => $workflow->toArray(),
            'packs' => $this->getRequiredPacks($workflow),
            'config' => $this->getConfig(),
            'environment' => $this->getEnvironment(),
        ]);
    }
}
```

## Security

### Authentication

```yaml
# remote-config.yml
remotes:
  production:
    type: ssh
    host: server.com
    user: deploy
    key: ~/.ssh/id_rsa
    knownHosts: ~/.ssh/known_hosts
```

### Secrets Management

```yaml
steps:
  - task: deploy
    secrets:
      API_KEY: "{{vault:api-key}}"
      DB_PASSWORD: "{{env:DB_PASSWORD}}"
```

### Sandboxing

Execute in isolated containers or VMs:

```yaml
remote:
  sandbox: true
  limits:
    cpu: "2"
    memory: "4GB"
    disk: "10GB"
    network: false    # Disable network access
```

## Features

### 1. Streaming Output

Stream remote execution output in real-time:

```bash
sir run workflow --remote server --follow
```

### 2. Artifact Transfer

Automatically transfer artifacts:

```yaml
steps:
  - task: build
    saveAs: build
    
  - task: deploy
    remote: production
    artifacts:
      - "{{build.path}}/dist"
```

### 3. Resource Management

Monitor and limit resource usage:

```yaml
remote:
  limits:
    timeout: 3600s
    cpu: 2
    memory: 4GB
  monitoring:
    enabled: true
    metrics: ["cpu", "memory", "network"]
```

### 4. Caching

Cache dependencies across remote executions:

```yaml
remote:
  cache:
    paths:
      - vendor/
      - node_modules/
    key: "{{checksum('composer.lock')}}"
```

## Implementation Steps

1. **Remote Executor Interface**
   - Define common interface
   - Implement SSH executor
   - Implement Docker executor

2. **Workflow Packaging**
   - Bundle workflow files
   - Include dependencies
   - Compress and transfer

3. **Security Layer**
   - Authentication mechanisms
   - Secrets management
   - Access control

4. **Monitoring & Logs**
   - Stream execution logs
   - Track resource usage
   - Handle failures

5. **CLI Integration**
   - `--remote` flag
   - Remote configuration
   - Status tracking

## Testing Strategy

- Unit tests for packaging logic
- Integration tests with local Docker
- E2E tests with SSH to test VM
- Security testing for isolation

## Future Enhancements

- **Multi-region execution**: Execute on nearest available region
- **Failover**: Automatic retry on different hosts
- **Load balancing**: Distribute across multiple hosts
- **Cost optimization**: Choose cheapest execution environment
