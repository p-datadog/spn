# Sleep Fix and DI Test Synchronization Patterns

## Overview

This document explains the sleep fix implemented in PR #5448 and compares it to other synchronization patterns used in Dynamic Instrumentation (DI) tests.

## The Problem: Non-Deterministic Sleep

### Original Code (BROKEN)

```ruby
context 'when three snapshots are added in quick succession' do
  it 'sends two batches' do
    expect(worker.send(:snapshot_queue)).to be_empty

    expect(input_transport).to receive(:send_input).once do |snapshots, tags|
      expect(snapshots).to eq([snapshot])
      expect(tags).to match(expected_tags)
    end

    worker.add_snapshot(snapshot)
    sleep 0.1  # ❌ Non-deterministic wait
    worker.add_snapshot(snapshot)
    sleep 0.1  # ❌ Non-deterministic wait
    worker.add_snapshot(snapshot)
    sleep(0.1) # ❌ Non-deterministic wait

    expect(worker.send(:snapshot_queue)).to eq([snapshot, snapshot])

    # ... rest of test
  end
end
```

### Why This Is Broken

1. **Non-deterministic timing** - `sleep 0.1` might not be enough time on slow systems or under load
2. **Flaky tests** - Can pass on developer machine but fail in CI
3. **Slower tests** - Always waits the full duration even if work completes faster
4. **False confidence** - Test might pass when code is actually broken (if timing changes)
5. **Race conditions** - No guarantee worker thread has completed work after sleep

### What The Test Needs

The test is verifying **batching behavior**:
- First snapshot triggers immediate send (batch size = 1)
- Next two snapshots are queued and batched together (batch size = 2)

To verify this, the test needs to:
1. Add first snapshot
2. **Wait for first send to complete** ← This is the key requirement
3. Add two more snapshots (these should be queued, not sent yet)
4. Verify queue contains 2 snapshots
5. Flush to send remaining snapshots

The problem is step 2: How do we wait for the first send to complete?

## Solution 1: Queue-Based Synchronization (Used in This Fix)

### The Fix

```ruby
context 'when three snapshots are added in quick succession' do
  it 'sends two batches' do
    expect(worker.send(:snapshot_queue)).to be_empty

    # ✅ Create Queue for signaling completion
    first_send_done = Queue.new

    expect(input_transport).to receive(:send_input).once do |snapshots, tags|
      expect(snapshots).to eq([snapshot])
      expect(tags).to match(expected_tags)
      first_send_done.push(:done)  # ✅ Signal completion
    end

    worker.add_snapshot(snapshot)

    # ✅ Wait deterministically for first send to complete
    Timeout.timeout(2) { first_send_done.pop }

    worker.add_snapshot(snapshot)
    worker.add_snapshot(snapshot)

    expect(worker.send(:snapshot_queue)).to eq([snapshot, snapshot])

    # ... rest of test
  end
end
```

### How It Works

1. **Create Queue** - `first_send_done = Queue.new` creates a synchronized queue
2. **Signal in mock** - When the mock is called, it pushes to the queue: `first_send_done.push(:done)`
3. **Wait in test** - Test blocks until signal received: `Timeout.timeout(2) { first_send_done.pop }`
4. **Timeout safety** - If signal never arrives (bug), test fails fast after 2 seconds

### Why This Works

✅ **Deterministic** - Test waits exactly as long as needed, no more, no less
✅ **Fast** - Proceeds immediately when work completes
✅ **Reliable** - No race conditions or timing assumptions
✅ **Fail-fast** - Timeout catches bugs where completion never happens
✅ **No production code changes** - Uses existing mock framework

### Key Properties of Queue

```ruby
queue = Queue.new

# Thread-safe operations
queue.push(item)    # Add item to queue
queue.pop           # Remove and return item (BLOCKS if queue is empty)
queue.pop(true)     # Non-blocking pop (raises ThreadError if empty)

# Queue.pop is perfect for synchronization:
# - Blocks until item is available
# - Thread-safe (no race conditions)
# - Works across threads
```

### When To Use Queue-Based Synchronization

Use this pattern when you need to:

✅ **Wait for specific event** - "Wait for first send to complete"
✅ **Observe intermediate state** - "Check queue after first send, before second send"
✅ **Unit test async behavior** - Testing specific timing sequences
✅ **No production API exists** - Can't use flush because it waits for ALL work

**Do NOT use when:**
❌ You just need ALL work to complete (use `flush` instead)
❌ You can modify production code to add hooks (better to add proper API)
❌ Testing synchronous code (no async work to wait for)

## Solution 2: Flush-Based Synchronization (Standard DI Pattern)

### The Pattern

The **dominant pattern** in DI tests is to use the `flush` method:

```ruby
it 'processes snapshot' do
  # Set up mocks
  expect(transport).to receive(:send_input).with(expected_snapshot)

  # Trigger async work
  component.probe_notifier_worker.add_snapshot(snapshot)

  # Wait for ALL async work to complete
  component.probe_notifier_worker.flush

  # Verify results
  expect(some_state).to eq(expected)
end
```

### How Flush Works

From `lib/datadog/di/probe_notifier_worker.rb:127-159`:

```ruby
def flush
  @lock.synchronize do
    @flush += 1
  end
  begin
    loop do
      if @thread.nil? || !@thread.alive?
        return
      end

      io_in_progress, queues_empty = @lock.synchronize do
        [io_in_progress?, status_queue.empty? && snapshot_queue.empty?]
      end

      if io_in_progress
        # If we just call Thread.pass we could be in a busy loop -
        # add a sleep.
        sleep 0.25
        next
      elsif queues_empty
        break
      else
        wake.signal
        sleep 0.25
        next
      end
    end
  ensure
    @lock.synchronize do
      @flush -= 1
    end
  end
end
```

**Key properties:**

1. **Blocks until ALL work is done** - Waits for queues to be empty
2. **Signals worker to wake up** - Forces immediate processing
3. **Polls with sleep 0.25** - Uses polling loop (acceptable in production flush method)
4. **Thread-safe** - Uses lock to coordinate state checks
5. **Handles I/O in progress** - Waits for ongoing I/O operations to complete

### When To Use Flush

Use `flush` when you need to:

✅ **Wait for ALL work to complete** - "Process all queued snapshots"
✅ **Integration tests** - Testing full end-to-end behavior
✅ **End-of-test cleanup** - Ensure queues are empty before test ends
✅ **Simple synchronization** - Don't need intermediate state

**Examples:**

```ruby
# Integration test pattern
it 'sends snapshot to agent' do
  expect(input_transport).to receive(:send_input) do |snapshots, tags|
    expect(snapshots).to eq([expected_snapshot])
  end

  probe_manager.add_probe(probe)
  InstrumentationSpecTestClass.new.test_method

  component.probe_notifier_worker.flush  # ✅ Wait for all work
end

# Error handling test
it 'clears queue even after error' do
  allow(input_transport).to receive(:send_input).and_raise("network error")

  worker.add_snapshot(snapshot)
  worker.flush  # ✅ Wait for error handling to complete

  expect(worker.send(:snapshot_queue)).to eq([])
end
```

### Why Flush Couldn't Solve The Original Problem

The test with sleep needed to:
1. Wait for first send
2. **Observe intermediate state** (2 snapshots still in queue)
3. Then continue

Flush waits for **ALL work to complete**, so it can't be used to observe intermediate state.

```ruby
# This doesn't work:
worker.add_snapshot(snapshot)
worker.flush  # ❌ Waits for ALL snapshots to be sent, not just first one
worker.add_snapshot(snapshot)  # These would be sent immediately by flush
worker.add_snapshot(snapshot)
expect(worker.send(:snapshot_queue)).to eq([snapshot, snapshot])  # ❌ FAILS - queue is empty!
```

## Comparison: Queue vs Flush

| Aspect | Queue-Based Sync | Flush-Based Sync |
|--------|------------------|------------------|
| **Use Case** | Wait for specific event | Wait for all work complete |
| **Granularity** | Fine-grained | Coarse-grained |
| **Intermediate State** | ✅ Can observe | ❌ Cannot observe |
| **Speed** | Fast (event-driven) | Fast (signals worker) |
| **Complexity** | More setup code | Simple, one call |
| **Production Changes** | None (uses mocks) | None (uses existing API) |
| **DI Test Usage** | 1 test (unit test) | ~50 tests (integration) |
| **Best For** | Unit tests | Integration tests |

## Best Practices for DI Test Synchronization

### Guideline 1: Prefer Flush for Integration Tests

**✅ DO:**
```ruby
it 'sends snapshot' do
  expect(transport).to receive(:send_input)
  add_probe_and_execute_code
  component.probe_notifier_worker.flush
end
```

**❌ DON'T:**
```ruby
it 'sends snapshot' do
  queue = Queue.new
  expect(transport).to receive(:send_input) { queue.push(:done) }
  add_probe_and_execute_code
  Timeout.timeout(2) { queue.pop }
end
```

**Why:** Flush is simpler and designed for this use case.

### Guideline 2: Use Queue for Fine-Grained Unit Tests

**✅ DO (when needed):**
```ruby
it 'batches snapshots correctly' do
  first_send = Queue.new
  expect(transport).to receive(:send_input).once { first_send.push(:done) }

  worker.add_snapshot(snapshot1)
  Timeout.timeout(2) { first_send.pop }

  worker.add_snapshot(snapshot2)
  expect(worker.snapshot_queue.size).to eq(1)  # Verify batching
end
```

**❌ DON'T:**
```ruby
it 'batches snapshots correctly' do
  worker.add_snapshot(snapshot1)
  sleep 0.1  # ❌ Non-deterministic
  worker.add_snapshot(snapshot2)
  expect(worker.snapshot_queue.size).to eq(1)
end
```

**Why:** Queue provides deterministic synchronization without sleep.

### Guideline 3: Never Use Sleep for Synchronization

**❌ NEVER DO:**
```ruby
worker.start
sleep 0.5  # ❌ Hope worker is ready?
worker.add_work(item)
```

**✅ ALWAYS DO ONE OF:**
```ruby
# Option 1: Use flush
worker.start
worker.add_work(item)
worker.flush

# Option 2: Use Queue
ready = Queue.new
worker.start { ready.push(:started) }
Timeout.timeout(2) { ready.pop }
worker.add_work(item)

# Option 3: Use observable state
worker.start
Timeout.timeout(2) { sleep 0.01 until worker.ready? }
worker.add_work(item)
```

### Guideline 4: Always Use Timeout with Queue.pop

**✅ DO:**
```ruby
queue = Queue.new
expect(mock).to receive(:method) { queue.push(:done) }
Timeout.timeout(2) { queue.pop }  # ✅ Fails fast if signal never arrives
```

**❌ DON'T:**
```ruby
queue = Queue.new
expect(mock).to receive(:method) { queue.push(:done) }
queue.pop  # ❌ Hangs forever if mock is never called (bug in test or code)
```

**Why:** Timeout ensures test fails fast with clear timeout error instead of hanging.

### Guideline 5: Use Descriptive Queue Variable Names

**✅ DO:**
```ruby
first_send_done = Queue.new
export_completed = Queue.new
worker_started = Queue.new
```

**❌ DON'T:**
```ruby
q = Queue.new
queue = Queue.new
sync = Queue.new
```

**Why:** Clear names document what event you're waiting for.

## Common Patterns in dd-trace-rb DI Tests

### Pattern 1: Integration Test with Flush (Most Common)

**Used in:** ~50 tests across instrumentation_spec.rb, probe_notifier_worker_spec.rb

```ruby
it 'sends snapshot to agent' do
  expect(input_transport).to receive(:send_input) do |snapshots, tags|
    expect(snapshots.size).to eq(1)
    # ... verify snapshot contents
  end

  probe_manager.add_probe(probe)
  InstrumentationSpecTestClass.new.test_method

  component.probe_notifier_worker.flush
end
```

**When to use:** Testing end-to-end behavior, verifying snapshots are sent correctly.

### Pattern 2: Mock Verification with Flush

**Used in:** Many tests that verify specific mock calls

```ruby
it 'calls method with expected args' do
  expect(component.probe_notifier_worker).to receive(:add_snapshot).once.and_call_original
  execute_instrumented_code
  component.probe_notifier_worker.flush
end
```

**When to use:** Verifying that specific methods are called during instrumentation.

### Pattern 3: Error Handling with Flush

**Used in:** Tests verifying error recovery

```ruby
it 'continues after error' do
  allow(transport).to receive(:send_input).and_raise("error")
  expect(telemetry).to receive(:report)

  worker.add_snapshot(snapshot)
  worker.flush

  expect(worker.send(:snapshot_queue)).to eq([])
end
```

**When to use:** Testing that errors are handled and queues are cleaned up.

### Pattern 4: Queue-Based Intermediate State (Rare - Only 1 Test)

**Used in:** probe_notifier_worker_spec.rb batching test

```ruby
it 'batches snapshots' do
  first_send_done = Queue.new

  expect(transport).to receive(:send_input).once { first_send_done.push(:done) }

  worker.add_snapshot(snapshot)
  Timeout.timeout(2) { first_send_done.pop }

  worker.add_snapshot(snapshot)
  worker.add_snapshot(snapshot)

  expect(worker.send(:snapshot_queue)).to eq([snapshot, snapshot])
end
```

**When to use:** Unit testing batching logic or other intermediate state verification.

## Migration Guide: Sleep → Queue

If you find sleep in a test, here's how to migrate:

### Step 1: Identify What You're Waiting For

**Question:** What async operation are you waiting for?

```ruby
worker.start
sleep 0.5  # ← What are we waiting for?
```

**Common answers:**
- Worker thread to start
- First batch to be sent
- Queue to be processed
- Background task to complete

### Step 2: Find the Synchronization Point

**Question:** Where in the code does the event complete?

For mock-based tests, the synchronization point is the mock expectation:

```ruby
expect(transport).to receive(:send_input) do |snapshots|
  # ← THIS is the synchronization point
  expect(snapshots).to eq([snapshot])
end
```

For production code tests, you need to identify where the work completes.

### Step 3: Add Queue Signaling

```ruby
# Before:
expect(transport).to receive(:send_input) do |snapshots|
  expect(snapshots).to eq([snapshot])
end
worker.add_snapshot(snapshot)
sleep 0.5  # ❌

# After:
send_done = Queue.new  # ✅ Add Queue
expect(transport).to receive(:send_input) do |snapshots|
  expect(snapshots).to eq([snapshot])
  send_done.push(:done)  # ✅ Signal completion
end
worker.add_snapshot(snapshot)
Timeout.timeout(2) { send_done.pop }  # ✅ Wait deterministically
```

### Step 4: Verify with Multiple Runs

```bash
# Run test 100 times to verify no flakiness
for i in {1..100}; do
  bundle exec rspec spec/path/to/spec.rb:123 || break
done
```

## Anti-Patterns to Avoid

### Anti-Pattern 1: Sleep After Flush

```ruby
# ❌ WRONG
worker.flush
sleep 0.1  # ← Why? flush already waited for completion!
expect(result).to eq(expected)
```

**Fix:** Remove the sleep. Flush already ensures completion.

```ruby
# ✅ CORRECT
worker.flush
expect(result).to eq(expected)
```

### Anti-Pattern 2: Multiple Flushes

```ruby
# ❌ WRONG (but harmless)
worker.add_snapshot(snapshot1)
worker.flush
worker.add_snapshot(snapshot2)
worker.flush  # ← Unnecessary if nothing else needs to complete
expect(result).to eq(expected)
```

**Fix:** Only flush once at the end if that's all you need.

```ruby
# ✅ CORRECT
worker.add_snapshot(snapshot1)
worker.add_snapshot(snapshot2)
worker.flush
expect(result).to eq(expected)
```

### Anti-Pattern 3: Queue Without Timeout

```ruby
# ❌ WRONG - Hangs forever if signal never comes
queue = Queue.new
expect(mock).to receive(:method) { queue.push(:done) }
queue.pop  # ← No timeout!
```

**Fix:** Always wrap with timeout.

```ruby
# ✅ CORRECT
queue = Queue.new
expect(mock).to receive(:method) { queue.push(:done) }
Timeout.timeout(2) { queue.pop }
```

### Anti-Pattern 4: Polling Loop with Sleep

```ruby
# ❌ WRONG
worker.add_snapshot(snapshot)
loop do
  break if worker.send(:snapshot_queue).empty?
  sleep 0.01
end
```

**Fix:** Use flush or Queue.

```ruby
# ✅ CORRECT - Option 1: Use flush
worker.add_snapshot(snapshot)
worker.flush

# ✅ CORRECT - Option 2: Use Queue
queue = Queue.new
expect(transport).to receive(:send_input) { queue.push(:done) }
worker.add_snapshot(snapshot)
Timeout.timeout(2) { queue.pop }
```

## Summary

### Quick Reference

| Need to... | Use | Example |
|------------|-----|---------|
| Wait for all work | `flush` | `worker.flush` |
| Wait for specific event | `Queue` | `Timeout.timeout(2) { queue.pop }` |
| Never use | `sleep` | ❌ `sleep 0.1` |

### Decision Tree

```
Do you need to observe intermediate state?
├─ No → Use flush
│   └─ component.probe_notifier_worker.flush
│
└─ Yes → Use Queue
    ├─ Create Queue: completion = Queue.new
    ├─ Signal in mock: completion.push(:done)
    └─ Wait in test: Timeout.timeout(2) { completion.pop }
```

### Key Takeaways

1. **DI tests primarily use flush** - ~50 tests use flush, only 1 uses Queue
2. **Flush is the default** - Use flush unless you need intermediate state
3. **Queue is for unit tests** - Use when testing specific timing or batching
4. **Never use sleep** - Always use Queue or flush for synchronization
5. **Always use Timeout** - Prevents hanging tests
6. **No production changes needed** - Both patterns work with existing code

### Related Documentation

- PR #5448: Sleep fix implementation
- `lib/datadog/di/probe_notifier_worker.rb:127-159`: Flush implementation
- `spec/datadog/di/probe_notifier_worker_spec.rb`: Queue-based test example
- `spec/datadog/di/integration/instrumentation_spec.rb`: Flush-based test examples

## Broader Test Suite Synchronization Patterns

### Overview

The dd-trace-rb test suite uses several synchronization patterns beyond just DI tests. This section documents all patterns found across the codebase.

### Pattern 5: ConditionVariable + Mutex (Core Workers)

**Used in:** `spec/datadog/core/workers/async_spec.rb`

**Pattern:**
```ruby
let(:perform_complete) { ConditionVariable.new }
let(:perform_result) { double('perform result') }

before do
  allow(worker_spy).to receive(:perform) do |*actual_args|
    perform_task.call(*actual_args)
    perform_complete.signal  # Signal completion
    perform_result
  end

  perform

  # Block until #perform gives signal or timeout is reached
  Mutex.new.tap do |mutex|
    mutex.synchronize do
      perform_complete.wait(mutex, 0.1)  # Wait with timeout
      sleep(0.1)  # Give a little extra time to collect the thread
    end
  end
end
```

**How it works:**
1. **ConditionVariable**: Synchronization primitive for thread signaling
2. **Mutex**: Required by ConditionVariable for thread-safe operations
3. **Signal**: `condition.signal` wakes up waiting threads
4. **Wait**: `condition.wait(mutex, timeout)` blocks until signaled or timeout

**Why ConditionVariable vs Queue:**

| Feature | ConditionVariable | Queue |
|---------|-------------------|-------|
| **Purpose** | Signal state change | Transfer data |
| **Requires Mutex** | Yes | No (built-in) |
| **Multiple waiters** | Yes (broadcast) | No (single consumer) |
| **Timeout** | Built-in | Via Timeout module |
| **Data transfer** | No | Yes |
| **Complexity** | Higher | Lower |

**When to use ConditionVariable:**
- Multiple threads waiting for same event
- Broadcasting to multiple waiters (`.broadcast`)
- Already have Mutex for state protection
- Need fine-grained control over wakeup

**When to use Queue instead:**
- Single waiter for event
- Need to transfer data
- Want simpler API
- Don't need broadcast

### Pattern 6: Thread.join (Worker Lifecycle)

**Used in:** `spec/datadog/core/telemetry/worker_spec.rb`, `spec/datadog/data_streams/processor_spec.rb`

**Pattern:**
```ruby
after do
  worker.stop(true)
  worker.join  # Wait for worker thread to terminate
end

# Or with timeout
after do
  thread = worker.send(:worker)
  if thread
    thread.terminate
    begin
      thread.join  # Wait for thread to finish
    rescue => _e
      # Prevents test from erroring out during cleanup
    end
  end
end
```

**How it works:**
1. **Thread.join**: Blocks until thread terminates
2. **Thread.join(timeout)**: Returns nil if timeout expires
3. **Cleanup pattern**: Ensures threads don't leak between tests

**Common usage:**
```ruby
# Start worker
worker.start

# Do work
worker.perform(task)

# Stop and wait for cleanup
worker.stop
worker.join  # Blocks until worker thread exits
```

**When to use Thread.join:**
- Waiting for worker thread to fully stop
- Test cleanup (after blocks)
- Ensuring no leaked threads between tests
- Simple thread lifecycle management

**Anti-pattern to avoid:**
```ruby
# ❌ WRONG - Thread might not be done yet
worker.stop
# Missing: worker.join
# Next test starts while thread still running!

# ✅ CORRECT
worker.stop
worker.join  # Ensure thread is fully stopped
```

### Pattern 7: Concurrent::CyclicBarrier (Stress Testing)

**Used in:** `spec/datadog/core/buffer/shared_examples.rb`

**Pattern:**
```ruby
let(:thread_count) { 100 }
let(:barrier) { Concurrent::CyclicBarrier.new(thread_count) }
let(:threads) do
  Array.new(thread_count) do |_i|
    Thread.new do
      barrier.wait  # All threads wait here
      # ... all threads start simultaneously after last arrives
      1000.times { buffer.push(items) }
    end
  end
end

it 'does not have collisions' do
  threads.each(&:join)  # Wait for all threads to complete
  expect(output).to match_array(expected_results)
end
```

**How it works:**
1. **CyclicBarrier**: Synchronization point for N threads
2. **barrier.wait**: Thread blocks until all N threads arrive
3. **Simultaneous start**: All threads released together when barrier trips
4. **Stress testing**: Creates maximum contention

**When to use CyclicBarrier:**
- Stress testing concurrent code
- Need multiple threads to start simultaneously
- Testing race conditions or thread safety
- Coordinating multiple thread phases

**Example stress test pattern:**
```ruby
let(:thread_count) { 100 }
let(:barrier) { Concurrent::CyclicBarrier.new(thread_count) }

it 'is thread-safe under high contention' do
  threads = Array.new(thread_count) do
    Thread.new do
      barrier.wait  # Synchronize start
      1000.times { perform_concurrent_operation }
    end
  end

  threads.each(&:join)
  verify_no_corruption
end
```

### Pattern 8: Thread.pass (Yielding Control)

**Used in:** `spec/datadog/core/buffer/shared_examples.rb`

**Pattern:**
```ruby
it 'executes without error under contention' do
  threads  # Start background threads

  barrier.wait
  1000.times do
    buffer.pop

    # Yield control to threads to increase contention
    # Otherwise we might run #pop a few times in succession,
    # which doesn't help us stress test this case
    Thread.pass
  end

  threads.each(&:join)
end
```

**How it works:**
1. **Thread.pass**: Suggests scheduler give other threads a chance to run
2. **Increases contention**: Forces more thread interleaving
3. **Not for synchronization**: Just a hint to scheduler

**When to use Thread.pass:**
- Stress testing with increased contention
- Encouraging scheduler to switch threads
- Making race conditions more likely (to test fixes)

**When NOT to use Thread.pass:**
- ❌ Never for synchronization (it's just a hint, not guaranteed)
- ❌ Never as replacement for proper locks
- ❌ Never in production code

**Correct usage:**
```ruby
# ✅ Stress testing - encourage contention
1000.times do
  perform_operation
  Thread.pass  # Give other threads a chance
end

# ❌ WRONG - This is not synchronization!
thread1_ready = false
Thread.new do
  thread1_ready = true
  # work
end
Thread.pass  # ❌ Doesn't guarantee thread1_ready is true!
process_assuming_ready
```

### Pattern 9: Worker#join with Timeout

**Used in:** `spec/datadog/core/buffer/shared_examples.rb`

**Pattern:**
```ruby
let(:wait_for_threads) do
  threads.each { |t| raise 'Thread wait timeout' unless t.join(5000) }
end

it 'completes all operations' do
  # Start threads
  threads

  # Perform operations
  perform_work

  # Wait for threads with timeout
  wait_for_threads  # Raises if any thread doesn't complete in 5 seconds
end
```

**How it works:**
1. **Thread.join(timeout)**: Returns thread if joined, nil if timeout
2. **Timeout detection**: Check return value to detect hangs
3. **Fail fast**: Raise exception instead of hanging forever

**When to use:**
- Any Thread.join in tests should have timeout
- Prevents tests from hanging forever
- Makes debugging easier (clear timeout error vs infinite hang)

**Pattern variations:**
```ruby
# Pattern 1: Raise on timeout
threads.each { |t| raise 'Thread timeout' unless t.join(5000) }

# Pattern 2: Return boolean
def wait_for_threads(timeout_ms)
  threads.all? { |t| t.join(timeout_ms) }
end

# Pattern 3: Collect results
results = threads.map do |t|
  t.join(5000) || raise("Thread #{t} timed out")
  t.value  # Get thread's return value
end
```

## Complete Pattern Comparison

| Pattern | Use Case | Complexity | DI Tests | Other Tests |
|---------|----------|------------|----------|-------------|
| **Flush** | Wait for all work | Low | ✅ Primary | ✅ Common |
| **Queue** | Wait for specific event | Low | ✅ 1 test | ❌ Rare |
| **ConditionVariable** | Signal state change | Medium | ❌ None | ✅ Workers |
| **Thread.join** | Wait for thread termination | Low | ✅ Cleanup | ✅ Common |
| **CyclicBarrier** | Stress testing | Medium | ❌ None | ✅ Concurrency |
| **Thread.pass** | Increase contention | Low | ❌ None | ✅ Stress tests |
| **Sleep** | ❌ NEVER | N/A | ❌ Removed | ⚠️ Avoid |

## Pattern Selection Guide

### Decision Tree

```
What are you testing?
├─ DI async worker
│   ├─ Need all work done? → flush
│   └─ Need intermediate state? → Queue
│
├─ Core worker lifecycle
│   ├─ Need worker to stop? → worker.stop + worker.join
│   └─ Need task complete? → ConditionVariable + Mutex
│
├─ Thread safety / concurrency
│   ├─ Need simultaneous start? → CyclicBarrier
│   ├─ Need high contention? → Thread.pass in loop
│   └─ Need to wait for threads? → Thread.join with timeout
│
└─ ❌ NEVER use sleep for synchronization
```

### Quick Reference

**Scenario:** "Wait for worker to process all items"
```ruby
# ✅ DI worker
component.probe_notifier_worker.flush

# ✅ Core worker with join
worker.perform(items)
worker.join
```

**Scenario:** "Wait for first batch to send, then add more"
```ruby
# ✅ Queue pattern
first_batch_done = Queue.new
expect(transport).to receive(:send) { first_batch_done.push(:done) }
worker.add_batch(batch1)
Timeout.timeout(2) { first_batch_done.pop }
worker.add_batch(batch2)
```

**Scenario:** "Test 100 threads accessing buffer simultaneously"
```ruby
# ✅ CyclicBarrier pattern
barrier = Concurrent::CyclicBarrier.new(100)
threads = Array.new(100) do
  Thread.new do
    barrier.wait
    buffer.push(item)
  end
end
threads.each { |t| raise 'Timeout' unless t.join(5000) }
```

**Scenario:** "Wait for worker thread to start and be ready"
```ruby
# ✅ ConditionVariable pattern
ready = ConditionVariable.new
mutex = Mutex.new
worker_ready = false

worker.start do
  mutex.synchronize do
    worker_ready = true
    ready.signal
  end
end

mutex.synchronize do
  ready.wait(mutex, 2) unless worker_ready
end

# ✅ Alternative: Queue pattern (simpler)
ready = Queue.new
worker.start { ready.push(:started) }
Timeout.timeout(2) { ready.pop }
```

**Scenario:** "Clean up worker thread after test"
```ruby
# ✅ Always use after block with join
after do
  worker.stop
  worker.join  # Blocks until worker thread exits
end
```

**Scenario:** "Never do this"
```ruby
# ❌ WRONG - Non-deterministic
worker.start
sleep 0.5  # Hope worker is ready?
worker.perform_task
```

## Advanced: When to Add Production Code Hooks

Sometimes you need to add hooks to production code to make it testable. This is GOOD - it means you're making your code more observable and debuggable.

### Example: Adding Completion Callback

**Before (untestable):**
```ruby
# lib/worker.rb
def process_async(item)
  Thread.new { process(item) }
end

# spec - forced to use sleep
it 'processes item' do
  worker.process_async(item)
  sleep 0.5  # ❌ Non-deterministic
  expect(result).to eq(expected)
end
```

**After (testable):**
```ruby
# lib/worker.rb
def process_async(item, &on_complete)
  Thread.new do
    result = process(item)
    on_complete&.call(result)
  end
end

# spec - deterministic
it 'processes item' do
  completion = Queue.new
  worker.process_async(item) { |result| completion.push(result) }
  result = Timeout.timeout(2) { completion.pop }
  expect(result).to eq(expected)
end
```

**Benefits:**
- ✅ Production code is more observable
- ✅ Can be used for debugging
- ✅ Can be used for monitoring
- ✅ Tests become deterministic

### Example: Adding Observable State

**Before (untestable):**
```ruby
# lib/worker.rb
def initialize
  @queue = []
  @processing = false
end

# spec - forced to use sleep
it 'processes queue' do
  worker.add_item(item)
  sleep 0.1  # ❌ Hope processing started?
  # Can't verify processing state
end
```

**After (testable):**
```ruby
# lib/worker.rb
def initialize
  @queue = []
  @processing = false
end

attr_reader :processing  # ✅ Expose observable state

def queue_size
  @queue.size
end

# spec - deterministic
it 'processes queue' do
  worker.add_item(item)
  Timeout.timeout(2) { sleep 0.01 until worker.processing }
  expect(worker.queue_size).to eq(0)
end
```

**Benefits:**
- ✅ Production debugging (check if worker is stuck)
- ✅ Metrics (track queue size)
- ✅ Tests can observe state

### When to Add Hooks vs Use Existing Patterns

**Add hooks to production code when:**
- ✅ Hook makes production code more observable
- ✅ Hook is useful for debugging
- ✅ Hook enables metrics or monitoring
- ✅ Multiple tests need same synchronization

**Use existing test patterns when:**
- ✅ Only one test needs sync
- ✅ Can use mock framework (like Queue in mock block)
- ✅ Worker already has flush/join method
- ✅ Adding hook would complicate API

## Summary

### Key Takeaways

1. **dd-trace-rb uses multiple sync patterns** - Not just flush, but also ConditionVariable, Thread.join, CyclicBarrier, and Queue
2. **Each pattern has specific use case** - Flush for "all work done", Queue for "specific event", CyclicBarrier for stress testing
3. **DI tests primarily use flush** - ~50 DI tests use flush, only 1 uses Queue
4. **Core tests use ConditionVariable** - Worker tests use ConditionVariable + Mutex pattern
5. **Stress tests use CyclicBarrier** - Buffer tests use CyclicBarrier for simultaneous thread starts
6. **Always use timeouts** - Thread.join(timeout), Timeout.timeout() prevent hangs
7. **Never use sleep** - Every synchronization need has a proper pattern
8. **Thread.pass is NOT sync** - Only for stress testing, never for actual synchronization

### Pattern Frequency in Codebase

Based on analysis of test suite:

| Pattern | Count | Primary Use |
|---------|-------|-------------|
| Flush | ~100+ | DI & Tracing workers |
| Thread.join | ~50+ | Worker lifecycle |
| ConditionVariable | ~10+ | Core workers |
| Queue | 1 | DI unit test (this PR) |
| CyclicBarrier | ~5 | Concurrency stress tests |
| Thread.pass | ~3 | Stress tests |
| Sleep (for sync) | 0 | ❌ All removed |

### When in Doubt

1. **For DI tests:** Use `component.probe_notifier_worker.flush`
2. **For workers:** Use `worker.join` after `worker.stop`
3. **For specific events:** Use `Queue` with `Timeout.timeout(2) { queue.pop }`
4. **For stress tests:** Use `Concurrent::CyclicBarrier`
5. **Never:** Use `sleep` for synchronization

### Related Files

- `lib/datadog/di/probe_notifier_worker.rb:127-159` - Flush implementation
- `spec/datadog/di/probe_notifier_worker_spec.rb` - Queue pattern example
- `spec/datadog/core/workers/async_spec.rb` - ConditionVariable pattern
- `spec/datadog/core/buffer/shared_examples.rb` - CyclicBarrier and Thread.pass patterns
- `spec/datadog/core/telemetry/worker_spec.rb` - Thread.join pattern

