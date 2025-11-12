# Backend & Node.js Deep Dive - Interview Answers

## Question 1: Explain Node.js event loop phases and how async/await works internally.

### Conceptual Explanation

The Node.js event loop is the mechanism that handles asynchronous operations. It has six main phases: **timers** (setTimeout/setInterval callbacks), **pending callbacks** (I/O callbacks deferred from previous iteration), **idle/prepare** (internal use), **poll** (retrieve new I/O events), **check** (setImmediate callbacks), and **close callbacks** (socket.close() events).

The poll phase is most important—it waits for I/O events and executes their callbacks. When the poll queue is empty, it checks if setImmediate callbacks are scheduled, then moves to check phase. If timers are ready, it wraps back to timers phase.

Async/await is syntactic sugar over Promises. When you `await` a Promise, the function execution pauses, the event loop continues processing other tasks, and when the Promise resolves, the function resumes with the resolved value. Under the hood, JavaScript uses microtasks queue (Promise callbacks) which has higher priority than macrotasks (setTimeout, I/O callbacks).

### Code Example

```javascript
// Event loop phases demonstration
console.log('1. Synchronous');

setTimeout(() => console.log('2. Timer phase'), 0);

setImmediate(() => console.log('3. Check phase'));

fs.readFile('file.txt', () => {
  console.log('4. I/O callback');
  
  setTimeout(() => console.log('5. Timer in I/O'), 0);
  setImmediate(() => console.log('6. setImmediate in I/O'));
});

Promise.resolve().then(() => console.log('7. Microtask'));

console.log('8. Synchronous end');

// Output order: 1, 8, 7, 2, 3, 4, 6, 5

// Async/await internals
async function fetchUser(userId) {
  // This pause allows event loop to process other tasks
  const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
  
  // Execution resumes here when Promise resolves
  return user;
}

// Equivalent Promise chain
function fetchUserPromise(userId) {
  return db.query('SELECT * FROM users WHERE id = $1', [userId])
    .then(user => user);
}

// Microtask vs Macrotask priority
console.log('Start');

setTimeout(() => console.log('Macrotask: setTimeout'), 0);

Promise.resolve().then(() => console.log('Microtask: Promise'));

console.log('End');

// Output: Start, End, Microtask: Promise, Macrotask: setTimeout
```

### Best Practices

- **Never block the event loop**: Avoid synchronous operations (fs.readFileSync, heavy CPU tasks) in production code as they prevent the event loop from processing other requests.
- **Use async/await for readability**: It makes asynchronous code look synchronous and easier to debug compared to Promise chains or callbacks.
- **Understand microtask priority**: Promise callbacks execute before setTimeout/setImmediate, which can help optimize task ordering.
- **Monitor event loop lag**: Use libraries like `event-loop-lag` to detect when the event loop is blocked, indicating performance issues.

---

## Question 2: How do you manage long-running or CPU-intensive tasks in Node.js?

### Conceptual Explanation

Node.js runs JavaScript on a single thread, so CPU-intensive tasks block the event loop and prevent handling other requests. Solutions include: **Worker Threads** for parallel CPU work within Node.js, **Child Processes** for running separate Node.js processes, **Job Queues** (Bull/BullMQ) for offloading work to background workers, and **External Services** for heavy computation (Python microservice, AWS Lambda).

Worker Threads share memory with the main thread and are efficient for parallelizing CPU tasks. Child Processes are fully isolated and better for running different programs. Job queues are ideal for tasks that can be processed asynchronously without blocking the API response.

### Code Example

```javascript
// Solution 1: Worker Threads for CPU-intensive tasks
const { Worker } = require('worker_threads');

app.post('/api/process-video', async (req, res) => {
  const { videoId } = req.body;
  
  // Return immediately, process in background
  res.json({ status: 'processing', videoId });
  
  // Run heavy task in worker thread
  const worker = new Worker('./workers/video-processor.js', {
    workerData: { videoId }
  });
  
  worker.on('message', (result) => {
    console.log('Video processed:', result);
    // Update database, send notification
  });
  
  worker.on('error', (error) => {
    console.error('Worker error:', error);
  });
});

// workers/video-processor.js
const { parentPort, workerData } = require('worker_threads');

function processVideo(videoId) {
  // CPU-intensive work here
  // This doesn't block the main thread
  const result = heavyVideoProcessing(videoId);
  return result;
}

const result = processVideo(workerData.videoId);
parentPort.postMessage(result);

// Solution 2: Bull Queue for background jobs
const Bull = require('bull');
const queue = new Bull('heavy-tasks');

// Add task to queue (non-blocking)
app.post('/api/generate-report', async (req, res) => {
  const job = await queue.add({ userId: req.user.id });
  res.json({ jobId: job.id, status: 'queued' });
});

// Process in separate worker process
queue.process(async (job) => {
  const report = await generateReport(job.data.userId);
  return report;
});
```

### Best Practices

- **Use job queues for async work**: If the user doesn't need immediate results, queue the task and return a job ID they can poll for status.
- **Worker threads for CPU tasks**: Image processing, data compression, cryptography—use worker threads to parallelize across CPU cores.
- **Set appropriate timeouts**: Long-running tasks should have timeouts to prevent infinite processing and resource leaks.
- **Monitor worker health**: Track worker failures, processing times, and queue depths to identify bottlenecks.

---

## Question 3: What's the difference between clustering and worker threads in Node.js?

### Conceptual Explanation

**Clustering** creates multiple Node.js processes (one per CPU core) that share the same server port. Each process has its own memory and event loop. The master process distributes incoming connections across workers using round-robin. This provides true isolation—if one worker crashes, others continue running. Best for scaling I/O-bound web servers.

**Worker Threads** run within a single process, sharing memory via SharedArrayBuffer and MessageChannel. They're lighter weight than processes and better for CPU-intensive tasks within a single application. Workers don't handle HTTP requests directly—the main thread does and delegates heavy computation to workers.

### Code Example

```javascript
// Clustering for horizontal scaling
const cluster = require('cluster');
const os = require('os');
const express = require('express');

if (cluster.isMaster) {
  const numCPUs = os.cpus().length;
  console.log(`Master process ${process.pid} starting ${numCPUs} workers`);
  
  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
  
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died. Forking new worker...`);
    cluster.fork(); // Restart crashed worker
  });
  
} else {
  // Worker process
  const app = express();
  
  app.get('/', (req, res) => {
    res.json({ pid: process.pid, message: 'Hello from worker' });
  });
  
  app.listen(3000, () => {
    console.log(`Worker ${process.pid} started`);
  });
}

// Worker Threads for CPU tasks
const { Worker } = require('worker_threads');

app.get('/api/calculate', (req, res) => {
  const worker = new Worker(`
    const { parentPort, workerData } = require('worker_threads');
    
    function fibonacci(n) {
      if (n <= 1) return n;
      return fibonacci(n - 1) + fibonacci(n - 2);
    }
    
    const result = fibonacci(workerData.number);
    parentPort.postMessage(result);
  `, { 
    eval: true,
    workerData: { number: 40 }
  });
  
  worker.on('message', (result) => {
    res.json({ result });
  });
});
```

### Best Practices

- **Use clustering for production web servers**: Utilize all CPU cores for handling concurrent HTTP requests. Tools like PM2 automate this.
- **Worker threads for computation**: Use when you need to perform CPU-heavy tasks without blocking the API response.
- **One cluster per server**: Clustering is process-level. In containerized environments (Kubernetes), run one Node process per container and scale containers instead.
- **Shared state requires coordination**: With clustering, each process has separate memory. Use Redis for shared cache/sessions across workers.

---

## Question 4: How would you structure a large-scale Node.js codebase for maintainability?

### Conceptual Explanation

Large codebases need clear structure to prevent them from becoming unmaintainable. Common patterns include **layered architecture** (routes → controllers → services → data access), **domain-driven design** (organize by business domains), and **clean architecture** (dependencies point inward toward business logic).

Key principles: separation of concerns (HTTP logic separate from business logic), dependency injection for testability, shared utilities in common modules, and consistent error handling. Use TypeScript for type safety in large teams. Structure folders by feature/domain rather than technical role when the codebase grows beyond ~20 files.

### Code Example

```typescript
// Feature-based structure
src/
├── modules/
│   ├── users/
│   │   ├── user.controller.ts
│   │   ├── user.service.ts
│   │   ├── user.repository.ts
│   │   ├── user.model.ts
│   │   └── user.routes.ts
│   ├── orders/
│   │   ├── order.controller.ts
│   │   ├── order.service.ts
│   │   ├── order.repository.ts
│   │   └── order.routes.ts
├── common/
│   ├── database/
│   ├── middleware/
│   ├── utils/
│   └── types/
├── config/
└── app.ts

// Layered architecture example

// 1. Routes (HTTP layer)
// users/user.routes.ts
import { Router } from 'express';
import { UserController } from './user.controller';

const router = Router();
const controller = new UserController();

router.get('/:id', controller.getUser);
router.post('/', controller.createUser);

export default router;

// 2. Controller (handles HTTP request/response)
// users/user.controller.ts
import { Request, Response } from 'express';
import { UserService } from './user.service';

export class UserController {
  private userService = new UserService();
  
  getUser = async (req: Request, res: Response) => {
    try {
      const user = await this.userService.getUserById(req.params.id);
      res.json(user);
    } catch (error) {
      res.status(500).json({ error: error.message });
    }
  };
  
  createUser = async (req: Request, res: Response) => {
    try {
      const user = await this.userService.createUser(req.body);
      res.status(201).json(user);
    } catch (error) {
      res.status(400).json({ error: error.message });
    }
  };
}

// 3. Service (business logic)
// users/user.service.ts
import { UserRepository } from './user.repository';
import { User, CreateUserDTO } from './user.model';

export class UserService {
  private userRepo = new UserRepository();
  
  async getUserById(id: string): Promise<User> {
    const user = await this.userRepo.findById(id);
    if (!user) throw new Error('User not found');
    return user;
  }
  
  async createUser(data: CreateUserDTO): Promise<User> {
    // Business logic: validate, transform, etc.
    if (!data.email.includes('@')) {
      throw new Error('Invalid email');
    }
    
    return this.userRepo.create(data);
  }
}

// 4. Repository (data access)
// users/user.repository.ts
import { pool } from '../../common/database';

export class UserRepository {
  async findById(id: string): Promise<User | null> {
    const result = await pool.query(
      'SELECT * FROM users WHERE id = $1',
      [id]
    );
    return result.rows[0] || null;
  }
  
  async create(data: CreateUserDTO): Promise<User> {
    const result = await pool.query(
      'INSERT INTO users (email, name) VALUES ($1, $2) RETURNING *',
      [data.email, data.name]
    );
    return result.rows[0];
  }
}
```

### Best Practices

- **Organize by feature/domain**: Group related files together (routes, controller, service, repository) rather than by type.
- **Keep business logic in services**: Controllers should be thin—just handle HTTP concerns. Services contain business rules and orchestrate data access.
- **Use dependency injection**: Makes code testable by allowing mock implementations. Consider using libraries like `tsyringe` or `inversify`.
- **Centralized error handling**: Use Express error middleware to handle errors consistently across the app.
- **Environment-based config**: Use environment variables for configuration. Libraries like `dotenv` and `joi` for validation help.

---

## Question 5: How do you handle error retries for async operations (like DB + file ops)?

### Conceptual Explanation

Transient errors (network timeouts, temporary database unavailability) should be retried with exponential backoff. Permanent errors (validation failures, not found) should not be retried. Implement retry logic with maximum attempts, increasing delays between retries, and circuit breakers to prevent overwhelming failing services.

Strategies include: manual retry loops with delays, libraries like `async-retry` or `p-retry`, and queue-based retries (Bull with built-in retry). Always log retry attempts and alert on exhausted retries. For critical operations, implement idempotency to safely retry without side effects.

### Code Example

```typescript
// Basic retry with exponential backoff
async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  let lastError: Error;
  
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      
      // Don't retry on certain errors
      if (error.code === 'NOT_FOUND' || error.code === 'VALIDATION_ERROR') {
        throw error;
      }
      
      if (attempt < maxRetries) {
        const delay = baseDelay * Math.pow(2, attempt); // Exponential: 1s, 2s, 4s
        console.log(`Retry attempt ${attempt + 1} after ${delay}ms`);
        await sleep(delay);
      }
    }
  }
  
  throw lastError;
}

// Usage
async function saveUserToDatabase(user: User) {
  return retryWithBackoff(async () => {
    return await db.query('INSERT INTO users (name, email) VALUES ($1, $2)', [user.name, user.email]);
  }, 3, 1000);
}

// Using p-retry library
import pRetry from 'p-retry';

async function fetchDataFromAPI(url: string) {
  return pRetry(
    async () => {
      const response = await fetch(url);
      if (!response.ok) throw new Error('API error');
      return response.json();
    },
    {
      retries: 5,
      minTimeout: 1000,
      maxTimeout: 10000,
      onFailedAttempt: (error) => {
        console.log(`Attempt ${error.attemptNumber} failed. ${error.retriesLeft} retries left`);
      }
    }
  );
}

// Bull queue with automatic retries
import Bull from 'bull';

const queue = new Bull('data-processing', {
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 2000
    }
  }
});

queue.process(async (job) => {
  // If this throws, Bull automatically retries with backoff
  const result = await processData(job.data);
  return result;
});

queue.on('failed', (job, err) => {
  if (job.attemptsMade >= job.opts.attempts) {
    console.error(`Job ${job.id} permanently failed:`, err);
    // Send alert, log to error tracking
  }
});
```

### Best Practices

- **Retry only transient errors**: Network timeouts, 503 errors, database connection issues. Don't retry 400/404/validation errors.
- **Use exponential backoff**: Prevents overwhelming a recovering service. Add jitter (randomness) to prevent thundering herd.
- **Set maximum retries**: Typically 3-5 attempts. After that, log the error and alert the team.
- **Implement idempotency**: Use unique IDs or tokens so retrying the same operation multiple times has the same effect as doing it once.
- **Monitor retry rates**: High retry rates indicate systemic issues. Alert when retry rates exceed thresholds.

---

## Question 6: How do you monitor performance and memory usage in Node.js apps?

### Conceptual Explanation

Node.js monitoring involves tracking: **event loop lag** (indicates blocking code), **memory usage** (heap size, GC pauses), **CPU usage**, **request latency**, and **error rates**. Tools include APM solutions (New Relic, DataDog), Prometheus metrics, and built-in Node.js profiling.

Memory leaks are common in Node.js—usually from event listeners not being removed, closures holding references, or caching without limits. Use heap snapshots and tools like `clinic.js` or Chrome DevTools to identify leaks. Monitor garbage collection frequency and duration—frequent GC indicates memory pressure.

### Code Example

```javascript
// Basic performance monitoring
const express = require('express');
const app = express();

// Event loop lag monitoring
const lag = require('event-loop-lag')(1000); // Check every second

setInterval(() => {
  const currentLag = lag();
  console.log(`Event loop lag: ${currentLag}ms`);
  
  if (currentLag > 100) {
    console.warn('Event loop is lagging! Blocking code detected.');
  }
}, 5000);

// Memory usage monitoring
setInterval(() => {
  const usage = process.memoryUsage();
  console.log({
    heapUsed: `${Math.round(usage.heapUsed / 1024 / 1024)}MB`,
    heapTotal: `${Math.round(usage.heapTotal / 1024 / 1024)}MB`,
    external: `${Math.round(usage.external / 1024 / 1024)}MB`,
    rss: `${Math.round(usage.rss / 1024 / 1024)}MB`
  });
}, 10000);

// Request timing middleware
app.use((req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`${req.method} ${req.path} - ${duration}ms - ${res.statusCode}`);
    
    // Alert on slow requests
    if (duration > 1000) {
      console.warn(`Slow request: ${req.method} ${req.path} took ${duration}ms`);
    }
  });
  
  next();
});

// Prometheus metrics
const promClient = require('prom-client');
const register = new promClient.Registry();

// Collect default metrics (CPU, memory, GC, event loop)
promClient.collectDefaultMetrics({ register });

// Custom metrics
const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_ms',
  help: 'Duration of HTTP requests in ms',
  labelNames: ['method', 'route', 'status_code'],
  registers: [register]
});

app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer();
  res.on('finish', () => {
    end({ method: req.method, route: req.route?.path || req.path, status_code: res.statusCode });
  });
  next();
});

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// Heap snapshot for memory leak analysis
app.get('/debug/heap-snapshot', (req, res) => {
  const v8 = require('v8');
  const fs = require('fs');
  
  const filename = `heap-${Date.now()}.heapsnapshot`;
  const snapshot = v8.writeHeapSnapshot(filename);
  
  res.json({ snapshot: filename });
});
```

### Best Practices

- **Use APM tools in production**: New Relic, DataDog, or Elastic APM provide comprehensive monitoring with minimal setup. They track requests, errors, database queries, and external API calls.
- **Monitor event loop lag**: This is the most important metric for Node.js. Lag > 100ms indicates blocking code that needs optimization.
- **Set memory limits**: Use `--max-old-space-size` flag to set heap limits. This prevents Node from consuming all server memory before crashing.
- **Track error rates**: Monitor exception rates and alert on spikes. Use error tracking tools like Sentry or Rollbar.
- **Regular performance testing**: Use tools like `autocannon` or `k6` for load testing. Identify bottlenecks before they hit production.
- **Profile in production carefully**: CPU profiling and heap snapshots can impact performance. Use them during low-traffic periods or on specific instances.

