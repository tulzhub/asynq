# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

Asynq is a Go library for distributed task queue processing backed by Redis. It provides a simple, reliable, and efficient way to queue tasks and process them asynchronously with workers.

## Development Commands

### Core Development

```bash
# Build the main library
go build -v ./...

# Run tests with race detection and coverage
go test -race -v -coverprofile=coverage.txt -covermode=atomic ./...

# Run benchmarks
go test -run=^$ -bench=. -loglevel=debug ./...

# Lint the code
golangci-lint run
# or via Makefile
make lint

# Generate protobuf files (if modifying proto files)
make proto
```

### Tools Module

```bash
# Build the CLI tool
cd tools && go build -v ./... && cd ..

# Test the tools module
cd tools && go test -race -v ./... && cd ..

# Install the CLI tool
go install github.com/tulzhub/asynq/tools/asynq@latest
```

### X Module (Extensions)

```bash
# Build extensions module
cd x && go build -v ./... && cd ..

# Test extensions module
cd x && go test -race -v ./... && cd ..
```

## Architecture Overview

### Core Components

**Task Flow**: Client enqueues tasks → Redis stores tasks → Server pulls and processes tasks → Workers execute handlers

**Main Types**:

- `Task` - Unit of work with type, payload, and options
- `Client` - Enqueues tasks to Redis queues
- `Server` - Processes tasks from queues using workers
- `ServeMux` - Maps task types to handlers (like http.ServeMux)

### Internal Architecture

**Broker Layer** (`internal/rdb/`): Redis operations and Lua scripts for atomic queue operations

**Core Processors**:

- `processor.go` - Pulls tasks from queues and executes them
- `scheduler.go` - Handles delayed/scheduled tasks
- `forwarder.go` - Moves tasks between queue states
- `recoverer.go` - Recovers tasks from crashed workers
- `janitor.go` - Cleanup operations for expired tasks
- `heartbeat.go` - Worker health monitoring
- `syncer.go` - Synchronizes server state

**Queue Management**:

- `aggregator.go` - Groups tasks for batch processing
- `subscriber.go` - Queue event notifications
- `periodic_task_manager.go` - Cron-like scheduled tasks

### Module Structure

**Main Package** (root): Core library with Client, Server, Task, and configuration types

**Internal Packages**:

- `internal/rdb/` - Redis database layer with Lua scripts
- `internal/base/` - Common types and broker interface
- `internal/proto/` - Protocol buffer definitions
- `internal/log/` - Structured logging utilities
- `internal/errors/` - Error types and handling
- `internal/timeutil/` - Time manipulation utilities
- `internal/context/` - Context utilities

**Tools** (`tools/`): CLI tool for monitoring and managing queues

**Extensions** (`x/`): Additional packages like metrics exporters and rate limiting

## Key Configuration Patterns

### Redis Connection

Uses `RedisConnOpt` interface with implementations for:

- Single Redis instance
- Redis Sentinel (high availability)
- Redis Cluster (not fully supported due to Lua scripts)

### Server Configuration

Main config options in `Config` struct:

- `Concurrency` - Number of worker goroutines
- `Queues` - Queue priority mapping
- `RetryDelayFunc` - Custom retry delay calculation
- `ErrorHandler` - Global error handling
- `StrictPriority` - Queue processing order
- `HealthCheckFunc` - Custom health checks

### Task Options

Tasks support various options:

- `MaxRetry(n)` - Retry limits
- `Queue(name)` - Target queue
- `Timeout(d)` - Processing timeout
- `Deadline(t)` - Absolute deadline
- `Unique(ttl)` - Deduplication
- `ProcessIn(d)` / `ProcessAt(t)` - Scheduling

## Testing Strategy

### Test Organization

- Tests are collocated with source files (`*_test.go`)
- Extensive use of table-driven tests
- Integration tests use Redis test broker (`internal/testbroker/`)
- Benchmark tests measure performance characteristics

### Test Execution

```bash
# Run all tests with race detection
go test -race ./...

# Run specific package tests
go test -race ./internal/rdb

# Run with verbose output and coverage
go test -race -v -cover ./...

# Run benchmarks only
go test -run=^$ -bench=. ./...
```

### Test Dependencies

Tests require Redis server running (Docker recommended):

```bash
docker run -p 6379:6379 redis:7
```

## External Dependencies

### Required

- `github.com/redis/go-redis/v9` - Redis client
- `github.com/google/uuid` - UUID generation
- `github.com/robfig/cron/v3` - Cron expression parsing
- `github.com/spf13/cast` - Type conversion utilities

### Development

- `github.com/google/go-cmp` - Deep comparison for tests
- `go.uber.org/goleak` - Goroutine leak detection
- `google.golang.org/protobuf` - Protocol buffer support

## CLI Tool Usage

The `asynq` CLI tool provides queue and task inspection:

```bash
# Real-time dashboard
asynq dash

# Queue statistics
asynq stats

# List/manage queues
asynq queue ls
asynq queue pause <queue>
asynq queue unpause <queue>

# Task operations
asynq task ls --queue=<queue> --state=<state>
asynq task cancel <task_id>
asynq task delete <task_id>
```

## Development Notes

### Redis Version Requirements

- Minimum Redis 4.0 (Lua script support)
- Redis 7.0+ recommended for latest features

### Go Version Support

- Last two Go versions supported (currently 1.22.x and 1.23.x)
- Uses Go modules for dependency management

### Concurrent Safety

- All public APIs are safe for concurrent use
- Internal components use proper synchronization
- Uses atomic operations for state management where appropriate
