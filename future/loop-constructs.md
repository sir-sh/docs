# Loop Constructs in Workflows

## Overview

Add support for repeating workflow steps over collections, similar to loops in programming languages or CI/CD systems.

## Proposed Syntax

### For Each

```yaml
steps:
  # Simple array iteration
  - shell: "echo Processing {{item}}"
    forEach: ["file1.txt", "file2.txt", "file3.txt"]
```

### For Each with Variable

```yaml
steps:
  - task: process.file
    forEach: "{{files}}"
    with:
      file: "{{item}}"
```

### For Each with Index

```yaml
steps:
  - shell: "echo {{index}}: {{item}}"
    forEach: "{{items}}"
```

### Nested Loops

```yaml
steps:
  - shell: "echo {{outer}} - {{inner}}"
    forEach: "{{categories}}"
    as: outer
    nested:
      - shell: "process {{outer}}/{{item}}"
        forEach: "{{items}}"
```

## Use Cases

### 1. Process Multiple Files

```yaml
description: "Convert images"

inputs:
  images:
    type: array
    required: true

steps:
  - task: image.convert
    forEach: "{{images}}"
    with:
      input: "{{item}}"
      output: "{{item}}.webp"
      format: "webp"
```

### 2. Deploy to Multiple Environments

```yaml
steps:
  - task: deploy.app
    forEach: ["staging", "production", "eu-west"]
    with:
      environment: "{{item}}"
      version: "{{version}}"
```

### 3. Run Tests in Parallel

```yaml
steps:
  - task: test.suite
    forEach: "{{testSuites}}"
    parallel: true
    with:
      suite: "{{item}}"
```

### 4. Matrix Builds

```yaml
description: "Test on multiple platforms"

steps:
  - task: test.run
    forEach:
      os: ["ubuntu", "macos", "windows"]
      node: ["16", "18", "20"]
    with:
      os: "{{item.os}}"
      nodeVersion: "{{item.node}}"
```

## Implementation

### Loop Executor

```php
class LoopExecutor
{
    public function executeLoop(array $step, array $items, Context $context): array
    {
        $results = [];
        $as = $step['as'] ?? 'item';
        
        foreach ($items as $index => $item) {
            // Create loop context
            $loopContext = clone $context;
            $loopContext->set($as, $item);
            $loopContext->set('index', $index);
            
            // Execute step
            $result = $this->executeStep($step, $loopContext);
            $results[] = $result;
        }
        
        return $results;
    }
}
```

### Workflow Runner Integration

```php
private function executeStep(array $step, Context $context): mixed
{
    // Check for loop
    if (isset($step['forEach'])) {
        $items = $this->resolveForEach($step['forEach'], $context);
        $executor = new LoopExecutor();
        return $executor->executeLoop($step, $items, $context);
    }
    
    // Normal execution
    // ...
}

private function resolveForEach($forEach, Context $context): array
{
    // If it's a template, resolve it
    if (is_string($forEach) && str_contains($forEach, '{{')) {
        $resolved = $context->resolveTemplate($forEach);
        $forEach = $context->get(trim($resolved, '{}'));
    }
    
    // Ensure it's an array
    if (!is_array($forEach)) {
        throw new \InvalidArgumentException('forEach must resolve to an array');
    }
    
    return $forEach;
}
```

## Advanced Features

### Parallel Execution

```yaml
steps:
  - task: heavy.task
    forEach: "{{items}}"
    parallel: true      # Execute in parallel
    maxWorkers: 4       # Limit concurrent executions
    with:
      data: "{{item}}"
```

Implementation:

```php
class ParallelLoopExecutor
{
    public function executeParallel(array $step, array $items, int $maxWorkers): array
    {
        $pool = new ProcessPool($maxWorkers);
        $results = [];
        
        foreach ($items as $item) {
            $pool->submit(function() use ($step, $item) {
                // Execute step with item
                return $this->executeStep($step, $item);
            });
        }
        
        return $pool->wait();
    }
}
```

### Loop Controls

```yaml
steps:
  - task: find.target
    forEach: "{{items}}"
    with:
      item: "{{item}}"
    breakOn: "{{result.found}}"     # Stop loop if condition met
    continueOn: "{{result.skip}}"   # Skip to next iteration
```

### Loop with Condition

```yaml
steps:
  - task: process
    forEach: "{{items}}"
    if: "{{item.enabled}}"    # Only process if condition met
    with:
      data: "{{item}}"
```

### Collect Results

```yaml
steps:
  - task: fetch.data
    forEach: "{{urls}}"
    with:
      url: "{{item}}"
    saveAs: results       # Array of all results
```

## YAML Schema

```yaml
step:
  forEach: string | array      # Collection to iterate
  as: string                   # Variable name (default: 'item')
  parallel: bool               # Execute in parallel
  maxWorkers: int              # Max concurrent executions
  breakOn: string              # Condition to stop loop
  continueOn: string           # Condition to skip iteration
```

## Examples

### Complex Example: Multi-Region Deployment

```yaml
description: "Deploy to multiple regions with health checks"

inputs:
  regions:
    type: array
    default: ["us-east", "us-west", "eu-central"]

steps:
  # Build once
  - task: build.app
    saveAs: buildResult
  
  # Deploy to each region
  - task: deploy.region
    forEach: "{{regions}}"
    parallel: true
    maxWorkers: 3
    with:
      region: "{{item}}"
      artifact: "{{buildResult.artifact}}"
    saveAs: deployments
  
  # Health check each deployment
  - task: health.check
    forEach: "{{deployments}}"
    with:
      endpoint: "{{item.endpoint}}"
    breakOn: "!{{result.healthy}}"  # Stop if any unhealthy
```

### Matrix Testing

```yaml
description: "Test matrix"

steps:
  - task: test.run
    forEach:
      python: ["3.8", "3.9", "3.10", "3.11"]
      os: ["ubuntu-latest", "macos-latest"]
    with:
      pythonVersion: "{{item.python}}"
      os: "{{item.os}}"
    saveAs: testResults
```

## Output Format

### Loop Results

```json
{
  "loopResults": [
    {
      "index": 0,
      "item": "value1",
      "result": { "success": true }
    },
    {
      "index": 1,
      "item": "value2",
      "result": { "success": true }
    }
  ],
  "totalIterations": 2,
  "successCount": 2,
  "failureCount": 0
}
```

## Error Handling

### Stop on First Error (Default)

```yaml
steps:
  - task: process
    forEach: "{{items}}"
    stopOnError: true    # Default behavior
```

### Continue on Error

```yaml
steps:
  - task: process
    forEach: "{{items}}"
    continueOnError: true
    # Collects all results, including errors
```

## Performance Considerations

### Batch Processing

```yaml
steps:
  - task: process.batch
    forEach: "{{items}}"
    batchSize: 10      # Process 10 items at a time
    with:
      items: "{{batch}}"
```

### Rate Limiting

```yaml
steps:
  - task: api.call
    forEach: "{{endpoints}}"
    rateLimit: "10/minute"    # Max 10 calls per minute
```

## Testing

```php
test('executes loop for each item', function () {
    $step = [
        'shell' => 'echo {{item}}',
        'forEach' => ['a', 'b', 'c']
    ];
    
    $executor = new LoopExecutor();
    $results = $executor->executeLoop($step, ['a', 'b', 'c'], new Context());
    
    expect($results)->toHaveCount(3);
});
```

## Documentation

- Add loop examples to workflow guide
- Document all loop features and options
- Show performance best practices
- Explain parallel execution trade-offs

## Future Extensions

- **Range loops**: `forEach: range(1, 10)`
- **While loops**: `while: "{{condition}}"`
- **Loop transformations**: `map`, `filter`, `reduce`
- **Retry logic**: `retry: 3` for failed iterations
