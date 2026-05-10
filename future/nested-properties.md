# Nested Property Access in Templates

## Overview

Currently, template expressions only support top-level variables: `{{variable}}`. This enhancement would add support for nested property access: `{{object.property.subproperty}}`.

## Current Limitation

### Current Behavior

```yaml
steps:
  - task: workspace.temp
    saveAs: workspace
  
  # ❌ This doesn't work
  - task: files.write
    with:
      path: "{{workspace.workspace}}/file.txt"
```

### Workaround

```yaml
steps:
  - task: workspace.temp
    saveAs: workspaceResult
  
  # ✓ Must flatten to top-level variable
  - shell: "export WORKSPACE={{workspaceResult.workspace}}"
  
  - task: files.write
    with:
      path: "$WORKSPACE/file.txt"
```

## Proposed Implementation

### Syntax

Support dot notation for nested access:

```yaml
{{variable.property}}
{{variable.nested.deep.property}}
{{array.0.field}}  # Array index access
```

### Template Resolution Engine

#### Current Implementation

```php
// In Context.php
public function resolveTemplate(string $template): string
{
    return preg_replace_callback('/\{\{([^}]+)\}\}/', function ($matches) {
        $expression = trim($matches[1]);
        // Only handles simple variables
        return (string)$this->get($expression, '');
    }, $template);
}
```

#### Proposed Implementation

```php
public function resolveTemplate(string $template): string
{
    return preg_replace_callback('/\{\{([^}]+)\}\}/', function ($matches) {
        $expression = trim($matches[1]);
        
        // Handle nested property access
        if (str_contains($expression, '.')) {
            return $this->resolveNestedProperty($expression);
        }
        
        // Handle functions (shellQuote, default)
        if (preg_match('/(\w+)\((.+)\)/', $expression, $funcMatches)) {
            return $this->resolveFunction($funcMatches[1], $funcMatches[2]);
        }
        
        // Simple variable
        return (string)$this->get($expression, '');
    }, $template);
}

private function resolveNestedProperty(string $path): string
{
    $parts = explode('.', $path);
    $value = $this->variables;
    
    foreach ($parts as $part) {
        // Handle array index [0] or numeric keys
        if (is_numeric($part)) {
            $part = (int)$part;
        }
        
        if (is_array($value) && array_key_exists($part, $value)) {
            $value = $value[$part];
        } elseif (is_object($value) && property_exists($value, $part)) {
            $value = $value->$part;
        } else {
            return '';  // Property not found
        }
    }
    
    return (string)$value;
}
```

### Function Support

Support functions with nested properties:

```yaml
# Shell quote a nested property
path: "{{shellQuote(workspace.workspace)}}"

# Default value for nested property
path: "{{default(config.output.path, '/tmp')}}"
```

### Array Access

Support array indexing:

```yaml
# Access first item in array
{{results.0.output}}

# Access named key in array
{{metadata.files.0.path}}
```

## Use Cases

### 1. Workspace Management

```yaml
steps:
  - task: workspace.temp
    with:
      prefix: "project"
    saveAs: ws
  
  - task: git.clone
    with:
      repo: "https://github.com/user/repo.git"
      folder: "{{ws.workspace}}/repo"  # ✓ Direct nested access
    saveAs: clone
  
  - task: files.write
    with:
      path: "{{clone.path}}/output.txt"  # ✓ Nested from previous step
      content: "Done"
```

### 2. Configuration Access

```yaml
# config.yml has nested structure
# config:
#   database:
#     host: localhost
#     port: 5432

steps:
  - shell: "psql -h {{config.database.host}} -p {{config.database.port}}"
```

### 3. Complex Data Structures

```yaml
steps:
  - task: api.request
    with:
      url: "https://api.example.com/data"
    saveAs: apiResult
  
  # Access nested API response
  - shell: "echo User: {{apiResult.data.user.name}}"
  - shell: "echo Email: {{apiResult.data.user.email}}"
```

## Edge Cases

### Handling Missing Properties

```yaml
# If workspace.subdir doesn't exist, return empty string
path: "{{workspace.subdir}}/file.txt"  # → "/file.txt"

# Or use default
path: "{{default(workspace.subdir, '/tmp')}}/file.txt"  # → "/tmp/file.txt"
```

### Escaping Dots

```yaml
# If a variable name contains a dot
# Use special syntax or escaping
value: "{{vars['key.with.dot']}}"
```

### Type Safety

```yaml
# If intermediate value is not array/object
# e.g., workspace = "string"
# Then workspace.property should return empty
value: "{{workspace.property}}"  # → ""
```

## Performance Considerations

### Caching

Cache resolved paths to avoid repeated lookups:

```php
private array $resolvedCache = [];

private function resolveNestedProperty(string $path): string
{
    if (isset($this->resolvedCache[$path])) {
        return $this->resolvedCache[$path];
    }
    
    $value = $this->doResolveNested($path);
    $this->resolvedCache[$path] = $value;
    
    return $value;
}
```

### Validation

Pre-validate templates before execution:

```php
public function validateTemplate(string $template): array
{
    $errors = [];
    preg_match_all('/\{\{([^}]+)\}\}/', $template, $matches);
    
    foreach ($matches[1] as $expression) {
        if (!$this->canResolve($expression)) {
            $errors[] = "Cannot resolve: {{$expression}}";
        }
    }
    
    return $errors;
}
```

## Implementation Checklist

- [ ] Update Context::resolveTemplate() to parse nested paths
- [ ] Implement resolveNestedProperty() method
- [ ] Support array index access
- [ ] Update function handlers (shellQuote, default) for nested paths
- [ ] Add template validation
- [ ] Add caching for performance
- [ ] Write comprehensive unit tests
- [ ] Update documentation with examples
- [ ] Add error messages for invalid paths
- [ ] Consider backwards compatibility

## Testing Strategy

### Unit Tests

```php
test('resolves nested property', function () {
    $context = new Context();
    $context->set('workspace', ['root' => '/tmp/ws', 'repo' => 'myrepo']);
    
    expect($context->resolveTemplate('{{workspace.root}}'))->toBe('/tmp/ws');
    expect($context->resolveTemplate('{{workspace.repo}}'))->toBe('myrepo');
});

test('resolves deeply nested property', function () {
    $context = new Context();
    $context->set('data', [
        'user' => [
            'profile' => [
                'name' => 'John'
            ]
        ]
    ]);
    
    expect($context->resolveTemplate('{{data.user.profile.name}}'))->toBe('John');
});

test('returns empty for missing nested property', function () {
    $context = new Context();
    $context->set('data', ['key' => 'value']);
    
    expect($context->resolveTemplate('{{data.missing.property}}'))->toBe('');
});

test('resolves array index', function () {
    $context = new Context();
    $context->set('items', ['first', 'second', 'third']);
    
    expect($context->resolveTemplate('{{items.0}}'))->toBe('first');
    expect($context->resolveTemplate('{{items.1}}'))->toBe('second');
});
```

### Integration Tests

Test in actual workflows to ensure real-world usage works correctly.

## Documentation Updates

- Update README examples to show nested access
- Add to template expression documentation
- Include in workflow YAML schema docs
- Add troubleshooting section for common issues

## Backwards Compatibility

Ensure existing workflows continue to work:
- Simple variables: `{{var}}` - No change
- Functions: `{{shellQuote(var)}}` - Enhanced to support nested
- Invalid syntax: Should fail gracefully with error message

## Future Extensions

- **Optional chaining**: `{{object?.property?.sub}}` - Return empty if any part is missing
- **Array methods**: `{{array.length}}`, `{{array.join(',')}}``
- **Filters**: `{{value|upper}}`, `{{text|trim}}`
- **Conditionals**: `{{if variable.exists}}...{{endif}}`
