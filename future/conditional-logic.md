# Conditional Logic in Workflows

## Overview

Add support for conditional execution of workflow steps based on runtime conditions, similar to CI/CD systems like GitHub Actions or GitLab CI.

## Proposed Syntax

### Simple If/Then

```yaml
steps:
  - task: tests.auto
    with:
      cwd: "."
    saveAs: testResult
  
  # Only run if tests passed
  - task: deploy.production
    if: "{{testResult.success}}"
    with:
      environment: "production"
```

### Conditions with Operators

```yaml
steps:
  # Run if branch is main
  - task: deploy.production
    if: "{{branch}} == 'main'"
  
  # Run if not a pull request
  - task: publish.npm
    if: "{{pr}} == false"
  
  # Run if tests passed AND branch is main
  - task: deploy.staging
    if: "{{testResult.success}} && {{branch}} == 'main'"
```

### Multiple Conditions

```yaml
steps:
  - shell: "echo 'Deploying'"
    if: |
      {{env}} == 'production' &&
      {{testResult.exitCode}} == 0 &&
      {{branch}} == 'main'
```

## Supported Operators

### Comparison
- `==` - Equals
- `!=` - Not equals
- `>` - Greater than
- `<` - Less than
- `>=` - Greater than or equal
- `<=` - Less than or equal

### Logical
- `&&` - AND
- `||` - OR
- `!` - NOT

### String
- `contains` - String contains
- `startsWith` - String starts with
- `endsWith` - String ends with
- `matches` - Regex match

### Existence
- `exists` - Variable exists
- `isEmpty` - Variable is empty

## Examples

### Conditional Deployment

```yaml
description: "Deploy based on branch"

inputs:
  branch:
    type: string
    required: true

steps:
  - task: tests.auto
    saveAs: tests
  
  - task: deploy.staging
    if: "{{branch}} == 'develop'"
    with:
      environment: "staging"
  
  - task: deploy.production
    if: "{{branch}} == 'main' && {{tests.success}}"
    with:
      environment: "production"
```

### Error Handling

```yaml
steps:
  - task: risky.operation
    saveAs: result
    continueOnError: true
  
  # Only run cleanup if operation failed
  - task: cleanup.resources
    if: "!{{result.success}}"
  
  # Send notification on failure
  - task: notify.slack
    if: "{{result.exitCode}} != 0"
    with:
      message: "Operation failed with code {{result.exitCode}}"
```

### Feature Flags

```yaml
steps:
  - task: feature.deploy
    if: "{{config.features.newUI}} == true"
    with:
      feature: "new-ui"
```

## Implementation

### Condition Evaluator

```php
class ConditionEvaluator
{
    public function __construct(private Context $context) {}
    
    public function evaluate(string $condition): bool
    {
        // Resolve variables in condition
        $resolved = $this->context->resolveTemplate($condition);
        
        // Parse and evaluate expression
        return $this->evaluateExpression($resolved);
    }
    
    private function evaluateExpression(string $expr): bool
    {
        // Handle logical operators
        if (str_contains($expr, '&&')) {
            [$left, $right] = explode('&&', $expr, 2);
            return $this->evaluateExpression(trim($left)) 
                && $this->evaluateExpression(trim($right));
        }
        
        if (str_contains($expr, '||')) {
            [$left, $right] = explode('||', $expr, 2);
            return $this->evaluateExpression(trim($left)) 
                || $this->evaluateExpression(trim($right));
        }
        
        // Handle comparison operators
        if (preg_match('/(.+)(==|!=|>=|<=|>|<)(.+)/', $expr, $matches)) {
            $left = trim($matches[1]);
            $operator = $matches[2];
            $right = trim($matches[3]);
            
            return $this->compare($left, $operator, $right);
        }
        
        // Boolean evaluation
        return $this->toBoolean($expr);
    }
}
```

### Workflow Runner Integration

```php
// In WorkflowRunner::executeStep()
private function executeStep(array $step, Context $context): mixed
{
    // Check condition before executing
    if (isset($step['if'])) {
        $evaluator = new ConditionEvaluator($context);
        if (!$evaluator->evaluate($step['if'])) {
            return ['skipped' => true, 'reason' => 'Condition not met'];
        }
    }
    
    // Execute step normally
    // ...
}
```

## YAML Schema Update

```yaml
step:
  # Existing properties
  method: string
  task: string
  shell: string
  with: object
  saveAs: string
  
  # New properties
  if: string              # Condition expression
  continueOnError: bool   # Don't fail workflow if step fails
  timeout: int            # Max execution time in seconds
```

## Complex Conditionals

### Multi-line Conditions

```yaml
- task: deploy
  if: |
    {{tests.passed}} == true &&
    ({{branch}} == 'main' || {{branch}} == 'release') &&
    {{env.CI}} == true
```

### Nested Conditions

```yaml
- task: notify
  if: |
    {{deploy.status}} == 'failed' &&
    (
      {{notify.email}} == true ||
      {{notify.slack}} == true
    )
```

## Error Messages

### Invalid Condition Syntax
```
✗ Step 2 failed: Invalid condition syntax
  Condition: {{branch} == 'main'
  Error: Unclosed template expression
```

### Undefined Variable
```
⚠ Step 2 skipped: Condition references undefined variable
  Condition: {{unknownVar}} == true
  Use default() to provide fallback values
```

## Testing Strategy

### Unit Tests

```php
test('evaluates simple equality', function () {
    $context = new Context();
    $context->set('branch', 'main');
    
    $evaluator = new ConditionEvaluator($context);
    expect($evaluator->evaluate("{{branch}} == 'main'"))->toBeTrue();
});

test('evaluates AND conditions', function () {
    $context = new Context();
    $context->set('tests', true);
    $context->set('branch', 'main');
    
    $evaluator = new ConditionEvaluator($context);
    expect($evaluator->evaluate("{{tests}} && {{branch}} == 'main'"))->toBeTrue();
});
```

### Integration Tests

Test full workflows with conditional steps to ensure proper execution flow.

## Documentation

- Add conditional examples to workflow guide
- Document all operators and functions
- Provide troubleshooting guide
- Show best practices for complex conditions

## Backwards Compatibility

- Steps without `if` property execute unconditionally (current behavior)
- Existing workflows continue to work without changes

## Future Extensions

- **else/elseif**: Alternative execution paths
- **switch/case**: Multiple conditions
- **Custom functions**: User-defined condition functions
- **Expression builder**: UI for building complex conditions
