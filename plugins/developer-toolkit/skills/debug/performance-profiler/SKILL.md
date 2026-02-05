---
name: performance-profiler
description: |
  Use this skill when investigating performance issues. Activate when the user has slow code,
  needs to find performance bottlenecks, wants to profile application performance, is optimizing
  response times, or investigating memory usage.
---

# Performance Profiler

Identify and resolve performance bottlenecks in your code.

## When to Use

- Application is slower than expected
- Looking for optimization opportunities
- Memory usage is growing unexpectedly
- Users report slow response times
- Preparing for load testing

## Performance Investigation Workflow

```
┌─────────────────────────────────────────────────────────────┐
│ 1. MEASURE - Baseline current performance                   │
├─────────────────────────────────────────────────────────────┤
│ 2. PROFILE - Identify where time is spent                   │
├─────────────────────────────────────────────────────────────┤
│ 3. ANALYZE - Understand why it's slow                       │
├─────────────────────────────────────────────────────────────┤
│ 4. OPTIMIZE - Make targeted improvements                    │
├─────────────────────────────────────────────────────────────┤
│ 5. VERIFY - Confirm improvement without regression          │
└─────────────────────────────────────────────────────────────┘
```

## Measurement Techniques

### Basic Timing

```typescript
// Simple timing
console.time('operation');
await slowOperation();
console.timeEnd('operation'); // operation: 1234ms

// More precise
const start = performance.now();
await slowOperation();
const duration = performance.now() - start;
console.log(`Took ${duration.toFixed(2)}ms`);
```

### Structured Performance Logging

```typescript
class PerformanceTracker {
  private marks = new Map<string, number>();

  mark(name: string): void {
    this.marks.set(name, performance.now());
  }

  measure(name: string, startMark: string): number {
    const start = this.marks.get(startMark);
    if (!start) throw new Error(`Mark ${startMark} not found`);

    const duration = performance.now() - start;
    console.log(`[PERF] ${name}: ${duration.toFixed(2)}ms`);
    return duration;
  }
}

// Usage
const perf = new PerformanceTracker();

perf.mark('start');
await fetchData();
perf.measure('fetchData', 'start');

perf.mark('afterFetch');
await processData();
perf.measure('processData', 'afterFetch');

perf.mark('afterProcess');
await saveResults();
perf.measure('saveResults', 'afterProcess');
```

## CPU Profiling

### Node.js Profiling

```bash
# Generate CPU profile
node --prof app.js
# Process the log
node --prof-process isolate-*.log > processed.txt

# Or use built-in profiler
node --inspect app.js
# Then connect Chrome DevTools to chrome://inspect
```

### Programmatic Profiling

```typescript
import { Session } from 'inspector';
import { writeFileSync } from 'fs';

async function profileOperation<T>(
  name: string,
  operation: () => Promise<T>
): Promise<T> {
  const session = new Session();
  session.connect();

  session.post('Profiler.enable');
  session.post('Profiler.start');

  const result = await operation();

  const profile = await new Promise<any>((resolve) => {
    session.post('Profiler.stop', (err, { profile }) => {
      resolve(profile);
    });
  });

  writeFileSync(`${name}.cpuprofile`, JSON.stringify(profile));
  session.disconnect();

  return result;
}

// Usage
await profileOperation('data-processing', async () => {
  return processLargeDataset(data);
});
// Open .cpuprofile in Chrome DevTools
```

## Memory Profiling

### Heap Snapshots

```typescript
import v8 from 'v8';
import { writeFileSync } from 'fs';

function takeHeapSnapshot(filename: string): void {
  const snapshot = v8.writeHeapSnapshot();
  console.log(`Heap snapshot written to ${snapshot}`);
}

// Comparative analysis
takeHeapSnapshot('before.heapsnapshot');
await suspectedLeakyOperation();
takeHeapSnapshot('after.heapsnapshot');
// Compare in Chrome DevTools
```

### Memory Usage Tracking

```typescript
function logMemoryUsage(label: string): void {
  const usage = process.memoryUsage();
  console.log(`[MEMORY] ${label}:`);
  console.log(`  Heap Used: ${(usage.heapUsed / 1024 / 1024).toFixed(2)} MB`);
  console.log(`  Heap Total: ${(usage.heapTotal / 1024 / 1024).toFixed(2)} MB`);
  console.log(`  RSS: ${(usage.rss / 1024 / 1024).toFixed(2)} MB`);
}

// Monitor over time
setInterval(() => logMemoryUsage('periodic'), 5000);
```

## Common Bottlenecks

### 1. N+1 Queries

```typescript
// BAD: N+1 queries
async function getUsersWithPosts() {
  const users = await db.query('SELECT * FROM users');  // 1 query

  for (const user of users) {
    // N queries (one per user!)
    user.posts = await db.query('SELECT * FROM posts WHERE user_id = ?', [user.id]);
  }

  return users;
}

// GOOD: Single query with join
async function getUsersWithPostsOptimized() {
  return db.query(`
    SELECT u.*, p.*
    FROM users u
    LEFT JOIN posts p ON p.user_id = u.id
  `);
}

// GOOD: Batch query
async function getUsersWithPostsBatch() {
  const users = await db.query('SELECT * FROM users');
  const userIds = users.map(u => u.id);

  const posts = await db.query(
    'SELECT * FROM posts WHERE user_id IN (?)',
    [userIds]
  );

  const postsByUser = groupBy(posts, 'user_id');
  return users.map(u => ({ ...u, posts: postsByUser[u.id] || [] }));
}
```

### 2. Synchronous Operations

```typescript
// BAD: Blocking the event loop
function processLargeFile(path: string) {
  const content = fs.readFileSync(path);  // Blocks!
  return JSON.parse(content);
}

// GOOD: Async
async function processLargeFileAsync(path: string) {
  const content = await fs.promises.readFile(path);
  return JSON.parse(content);
}

// GOOD: Streaming for very large files
async function* processLargeFileStream(path: string) {
  const stream = fs.createReadStream(path);
  for await (const chunk of stream) {
    yield processChunk(chunk);
  }
}
```

### 3. Unnecessary Computation

```typescript
// BAD: Recalculating on every render
function Component({ items }) {
  // Runs on EVERY render
  const sorted = items.sort((a, b) => a.name.localeCompare(b.name));
  const filtered = sorted.filter(i => i.active);

  return <List items={filtered} />;
}

// GOOD: Memoized
function ComponentOptimized({ items }) {
  const processedItems = useMemo(() => {
    return items
      .filter(i => i.active)
      .sort((a, b) => a.name.localeCompare(b.name));
  }, [items]);

  return <List items={processedItems} />;
}
```

### 4. Memory Leaks

```typescript
// BAD: Event listener leak
function setupHandler() {
  window.addEventListener('resize', handleResize);
  // Never removed!
}

// GOOD: Cleanup
function setupHandlerWithCleanup() {
  window.addEventListener('resize', handleResize);

  return () => {
    window.removeEventListener('resize', handleResize);
  };
}

// React effect with cleanup
useEffect(() => {
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);
```

### 5. Large Bundle Size

```bash
# Analyze bundle
npx webpack-bundle-analyzer stats.json

# Or for Vite
npx vite-bundle-visualizer
```

```typescript
// BAD: Import entire library
import _ from 'lodash';
const result = _.pick(obj, ['a', 'b']);

// GOOD: Import specific function
import pick from 'lodash/pick';
const result = pick(obj, ['a', 'b']);

// BETTER: Use native
const result = { a: obj.a, b: obj.b };
```

## Optimization Strategies

### Caching

```typescript
const cache = new Map<string, { data: any; expires: number }>();

async function cachedFetch<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttlMs: number = 60000
): Promise<T> {
  const cached = cache.get(key);

  if (cached && cached.expires > Date.now()) {
    return cached.data;
  }

  const data = await fetcher();
  cache.set(key, { data, expires: Date.now() + ttlMs });
  return data;
}
```

### Batching

```typescript
class RequestBatcher<T> {
  private pending = new Map<string, Promise<T>>();
  private batch: string[] = [];
  private timeout: NodeJS.Timeout | null = null;

  async get(id: string): Promise<T> {
    if (this.pending.has(id)) {
      return this.pending.get(id)!;
    }

    const promise = new Promise<T>((resolve) => {
      this.batch.push(id);

      if (!this.timeout) {
        this.timeout = setTimeout(() => this.flush(), 10);
      }
    });

    this.pending.set(id, promise);
    return promise;
  }

  private async flush(): Promise<void> {
    const ids = [...this.batch];
    this.batch = [];
    this.timeout = null;

    const results = await this.batchFetch(ids);
    // Resolve all pending promises
  }
}
```

### Lazy Loading

```typescript
// React lazy loading
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <HeavyComponent />
    </Suspense>
  );
}
```

## Performance Checklist

- [ ] Measure before optimizing
- [ ] Profile to find actual bottlenecks
- [ ] Check for N+1 queries
- [ ] Review async operations
- [ ] Look for memory leaks
- [ ] Analyze bundle size
- [ ] Consider caching opportunities
- [ ] Test with realistic data volumes
- [ ] Verify improvement with benchmarks
