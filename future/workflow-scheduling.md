# Workflow Scheduling

## Overview

Add support for scheduled/automated workflow execution, allowing workflows to run at specific times or intervals without manual invocation.

## Use Cases

### 1. Automated Backups
Run backup workflows nightly at 2 AM.

### 2. Monitoring & Health Checks
Check service health every 5 minutes.

### 3. Report Generation
Generate weekly reports every Monday.

### 4. Cleanup Tasks
Remove old files daily.

## Proposed Syntax

### Cron-style Scheduling

```yaml
# In .sir/schedules.yml
schedules:
  - name: "nightly-backup"
    workflow: backup
    cron: "0 2 * * *"    # Every day at 2 AM
    inputs:
      target: "production"
  
  - name: "health-check"
    workflow: monitor
    cron: "*/5 * * * *"  # Every 5 minutes
    enabled: true
```

### Natural Language

```yaml
schedules:
  - name: "weekly-report"
    workflow: generate-report
    schedule: "every monday at 9am"
    inputs:
      format: "pdf"
```

### Event-based

```yaml
schedules:
  - name: "on-file-change"
    workflow: process
    watch: "/data/*.csv"
    inputs:
      file: "{{event.file}}"
```

## CLI Commands

```bash
# List scheduled workflows
sir schedules:list

# Add a schedule
sir schedules:add backup --cron "0 2 * * *"

# Remove a schedule
sir schedules:remove backup

# Run scheduler daemon
sir schedules:run

# Test a schedule (dry-run)
sir schedules:test backup
```

## Implementation

### Schedule Manager

```php
class ScheduleManager
{
    public function loadSchedules(): array
    {
        // Load from .sir/schedules.yml
        $layers = $this->layerResolver->resolve();
        $schedules = [];
        
        foreach ($layers as $layer) {
            $file = $layer . '/schedules.yml';
            if (file_exists($file)) {
                $layerSchedules = Yaml::parseFile($file);
                $schedules = array_merge($schedules, $layerSchedules['schedules'] ?? []);
            }
        }
        
        return $schedules;
    }
    
    public function shouldRun(array $schedule): bool
    {
        $cron = new CronExpression($schedule['cron']);
        return $cron->isDue();
    }
}
```

### Scheduler Daemon

```php
class SchedulerDaemon
{
    public function run(): void
    {
        while (true) {
            $schedules = $this->manager->loadSchedules();
            
            foreach ($schedules as $schedule) {
                if ($this->manager->shouldRun($schedule)) {
                    $this->executeSchedule($schedule);
                }
            }
            
            sleep(60); // Check every minute
        }
    }
    
    private function executeSchedule(array $schedule): void
    {
        try {
            $this->logger->info("Running scheduled workflow: {$schedule['name']}");
            
            $workflow = $this->loader->load($schedule['workflow']);
            $result = $this->runner->run($workflow, $schedule['inputs'] ?? []);
            
            $this->logger->info("Completed: {$schedule['name']}");
        } catch (\Exception $e) {
            $this->logger->error("Failed: {$schedule['name']}: " . $e->getMessage());
        }
    }
}
```

## Features

### 1. Execution History

Track schedule execution history:

```bash
sir schedules:history backup

┌────────────┬──────────────────────┬─────────┬──────────┐
│ Date       │ Time                 │ Status  │ Duration │
├────────────┼──────────────────────┼─────────┼──────────┤
│ 2026-01-12 │ 02:00:15            │ Success │ 45s      │
│ 2026-01-11 │ 02:00:10            │ Success │ 42s      │
│ 2026-01-10 │ 02:00:12            │ Failed  │ 5s       │
└────────────┴──────────────────────┴─────────┴──────────┘
```

### 2. Notifications

Send notifications on success/failure:

```yaml
schedules:
  - name: "critical-backup"
    workflow: backup
    cron: "0 2 * * *"
    notifications:
      onSuccess:
        - email: admin@example.com
        - slack: "#ops"
      onFailure:
        - email: admin@example.com
        - pagerduty: service-id
```

### 3. Concurrency Control

Prevent overlapping executions:

```yaml
schedules:
  - name: "long-process"
    workflow: process
    cron: "0 * * * *"
    concurrency: prevent  # Skip if previous run still active
    timeout: 3600s
```

### 4. Retry Logic

Automatically retry failed schedules:

```yaml
schedules:
  - name: "flaky-task"
    workflow: task
    cron: "*/15 * * * *"
    retry:
      attempts: 3
      backoff: exponential
      delay: 60s
```

## Schedule Configuration

### Full Schema

```yaml
schedules:
  - name: string              # Required: Unique schedule name
    workflow: string          # Required: Workflow to execute
    cron: string              # Cron expression
    schedule: string          # Natural language (alternative to cron)
    enabled: bool             # Enable/disable schedule
    inputs: object            # Workflow inputs
    concurrency: string       # prevent|allow|queue
    timeout: string           # Max execution time
    retry:
      attempts: int
      backoff: string
      delay: string
    notifications:
      onSuccess: array
      onFailure: array
    conditions:              # Additional conditions
      branch: string
      tag: string
```

## System Integration

### Systemd Service

```ini
# /etc/systemd/system/sir-scheduler.service
[Unit]
Description=Sir.sh Scheduler
After=network.target

[Service]
Type=simple
User=sir
WorkingDirectory=/home/sir
ExecStart=/usr/local/bin/sir schedules:run
Restart=always

[Install]
WantedBy=multi-user.target
```

### Docker Container

```dockerfile
FROM php:8.2-cli
COPY . /app
WORKDIR /app
CMD ["php", "application", "schedules:run"]
```

### Cron Integration

Alternative: Use system cron instead of daemon:

```bash
# Add to crontab
0 2 * * * cd /path/to/project && sir run backup
```

## Monitoring

### Health Endpoint

```bash
sir schedules:health

Scheduler Status: Running
Active Schedules: 5
Last Check: 2026-01-12 14:30:00
Next Runs:
  - backup: 2026-01-13 02:00:00
  - health-check: 2026-01-12 14:35:00
```

### Metrics

Track scheduler metrics:
- Total schedules
- Success/failure rates
- Average execution time
- Last run times

## Testing

```php
test('schedule runs at correct time', function () {
    $schedule = [
        'name' => 'test',
        'workflow' => 'test-workflow',
        'cron' => '* * * * *'  // Every minute
    ];
    
    $manager = new ScheduleManager();
    expect($manager->shouldRun($schedule))->toBeTrue();
});
```

## Documentation

- Schedule configuration guide
- Cron expression reference
- Notification setup instructions
- Troubleshooting common issues

## Future Enhancements

- **Web UI**: Visual schedule management
- **Distributed scheduling**: Run across multiple nodes
- **Dynamic schedules**: Adjust timing based on load
- **Schedule dependencies**: Chain scheduled workflows
- **Calendar integration**: Skip on holidays
