# Performance Guide

## Go Performance Patterns

### Concurrency with Worker Pools

```go
func processBatch(items []Item) []Result {
    numWorkers := 4
    jobs := make(chan Item, len(items))
    results := make(chan Result, len(items))

    // Start workers
    for w := 0; w < numWorkers; w++ {
        go func() {
            for item := range jobs {
                results <- processItem(item)
            }
        }()
    }

    // Send jobs
    for _, item := range items {
        jobs <- item
    }
    close(jobs)

    // Collect results
    var allResults []Result
    for i := 0; i < len(items); i++ {
        allResults = append(allResults, <-results)
    }

    return allResults
}
```

### Resource Limiting with Semaphores

```go
var semaphore = make(chan struct{}, 2) // Max 2 concurrent

func processLarge(data []byte) error {
    semaphore <- struct{}{}        // Acquire
    defer func() { <-semaphore }() // Release

    // Process large resource
    return nil
}
```

### Profiling

```bash
# CPU profiling
go test -cpuprofile=cpu.prof
go tool pprof cpu.prof

# Memory profiling
go test -memprofile=mem.prof
go tool pprof mem.prof
```
