# Scalability, Reliability, and Infrastructure - Interview Answers

## Question 1: How would you deploy and scale your app on Kubernetes or Docker?

### Conceptual Explanation

Docker containerizes applications for consistent deployment across environments. Kubernetes orchestrates containers at scale, handling deployment, scaling, load balancing, self-healing, and rolling updates. Core concepts: **Pods** (one or more containers), **Deployments** (manage pod replicas), **Services** (networking/load balancing), **ConfigMaps/Secrets** (configuration), and **Ingress** (external access).

Scaling strategies include **horizontal scaling** (add more pod replicas) using Horizontal Pod Autoscaler (HPA) based on CPU/memory/custom metrics, and **vertical scaling** (increase pod resources). Use **readiness probes** to ensure traffic only goes to healthy pods and **liveness probes** to restart crashed pods.

For Node.js apps, considerations include: single process per container (let Kubernetes handle scaling), proper signal handling (SIGTERM for graceful shutdown), health check endpoints, and using init containers for migrations or setup tasks.

### Code Example

```dockerfile
# Dockerfile for Node.js app
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .

# Multi-stage build for smaller image
FROM node:18-alpine

WORKDIR /app
COPY --from=builder /app .

# Run as non-root user
USER node

EXPOSE 3000

# Graceful shutdown handling
CMD ["node", "server.js"]
```

```yaml
# Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:v1.0.0
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: myapp-config
              key: database-host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: database-password
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5

---
# Service for load balancing
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: ClusterIP

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

---
# Ingress for external access
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

```javascript
// Graceful shutdown handling in Node.js
const express = require('express');
const app = express();

let isShuttingDown = false;

// Health check endpoints
app.get('/health', (req, res) => {
  if (isShuttingDown) {
    return res.status(503).json({ status: 'shutting down' });
  }
  res.json({ status: 'healthy' });
});

app.get('/ready', async (req, res) => {
  // Check if app is ready (DB connected, etc.)
  try {
    await db.query('SELECT 1');
    res.json({ status: 'ready' });
  } catch (error) {
    res.status(503).json({ status: 'not ready' });
  }
});

const server = app.listen(3000, () => {
  console.log('Server started on port 3000');
});

// Graceful shutdown on SIGTERM
process.on('SIGTERM', () => {
  console.log('SIGTERM received, starting graceful shutdown');
  isShuttingDown = true;
  
  // Stop accepting new connections
  server.close(() => {
    console.log('Server closed');
    
    // Close database connections
    db.end(() => {
      console.log('Database connections closed');
      process.exit(0);
    });
  });
  
  // Force shutdown after 30 seconds
  setTimeout(() => {
    console.error('Forced shutdown after timeout');
    process.exit(1);
  }, 30000);
});
```

### Best Practices

- **Use multi-stage builds**: Reduces final image size by excluding build tools and dev dependencies. Faster pulls and better security.
- **Set resource limits**: Prevents pods from consuming excessive resources. Requests for scheduling, limits for enforcement.
- **Implement health checks**: Readiness probes prevent traffic to unhealthy pods. Liveness probes restart stuck containers.
- **Graceful shutdown**: Handle SIGTERM properly—finish in-flight requests, close connections cleanly. Kubernetes gives 30s before SIGKILL.
- **Use HPA for auto-scaling**: Automatically scale based on load. Monitor metrics to tune min/max replicas and target utilization.
- **Don't store state in containers**: Use external databases, Redis, S3 for persistence. Containers are ephemeral.
- **Use namespaces**: Isolate environments (dev, staging, prod) and teams using Kubernetes namespaces.

---

## Question 2: How do you handle logging, monitoring, and alerting (ELK, Prometheus, etc.)?

### Conceptual Explanation

Observability has three pillars: **logs** (detailed events), **metrics** (numerical measurements), and **traces** (request flow). Logging solutions like ELK Stack (Elasticsearch, Logstash, Kibana) or EFK (Fluentd) collect, store, and visualize logs. Metrics systems like Prometheus collect time-series data with Grafana for visualization. Distributed tracing (Jaeger, Zipkin) tracks requests across microservices.

Best practices: **structured logging** (JSON format), **log levels** (error, warn, info, debug), **correlation IDs** (track requests across services), **centralized logging** (all services to one place), and **appropriate retention** (30-90 days for logs, 1+ year for metrics).

Alerting should be actionable—alert on symptoms (high error rate, slow response time) not causes (high CPU). Use alert fatigue prevention: proper thresholds, deduplication, and on-call rotation.

### Code Example

```javascript
// Structured logging with Winston
const winston = require('winston');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'myapp' },
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

// Request logging with correlation ID
const { v4: uuidv4 } = require('uuid');

app.use((req, res, next) => {
  req.id = req.headers['x-request-id'] || uuidv4();
  res.setHeader('X-Request-ID', req.id);
  
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    logger.info({
      message: 'HTTP Request',
      requestId: req.id,
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      duration,
      userAgent: req.get('user-agent'),
      userId: req.user?.id
    });
  });
  
  next();
});

// Structured error logging
app.use((err, req, res, next) => {
  logger.error({
    message: err.message,
    stack: err.stack,
    requestId: req.id,
    method: req.method,
    path: req.path,
    userId: req.user?.id
  });
  
  res.status(500).json({ error: 'Internal server error', requestId: req.id });
});

// Prometheus metrics
const promClient = require('prom-client');
const register = new promClient.Registry();

// Collect default metrics
promClient.collectDefaultMetrics({ register });

// Custom metrics
const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.5, 1, 2, 5],
  registers: [register]
});

const httpRequestTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  registers: [register]
});

// Metrics middleware
app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer();
  
  res.on('finish', () => {
    const route = req.route?.path || req.path;
    const labels = { method: req.method, route, status_code: res.statusCode };
    
    end(labels);
    httpRequestTotal.inc(labels);
  });
  
  next();
});

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// Distributed tracing with OpenTelemetry
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { registerInstrumentations } = require('@opentelemetry/instrumentation');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');

const provider = new NodeTracerProvider();
provider.register();

registerInstrumentations({
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation()
  ]
});
```

```yaml
# Prometheus configuration
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'myapp'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: myapp
      - source_labels: [__meta_kubernetes_pod_ip]
        target_label: __address__
        replacement: $1:3000

# Prometheus alerts
groups:
  - name: myapp_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status_code=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} req/s"
      
      - alert: HighResponseTime
        expr: histogram_quantile(0.95, http_request_duration_seconds_bucket) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time"
          description: "95th percentile is {{ $value }}s"
      
      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod is crash looping"
```

```yaml
# Fluentd configuration for log aggregation
<source>
  @type tail
  path /var/log/containers/*.log
  pos_file /var/log/fluentd-containers.log.pos
  tag kubernetes.*
  read_from_head true
  <parse>
    @type json
    time_format %Y-%m-%dT%H:%M:%S.%NZ
  </parse>
</source>

<filter kubernetes.**>
  @type kubernetes_metadata
  @id filter_kube_metadata
</filter>

<match kubernetes.**>
  @type elasticsearch
  host elasticsearch.logging.svc.cluster.local
  port 9200
  logstash_format true
  logstash_prefix kubernetes
  include_tag_key true
</match>
```

### Best Practices

- **Use structured JSON logs**: Makes logs searchable and parseable. Include requestId, userId, timestamp, level, message.
- **Implement correlation IDs**: Track requests across multiple services. Pass X-Request-ID header between services.
- **Log at appropriate levels**: ERROR for failures requiring attention, WARN for recoverable issues, INFO for business events, DEBUG for troubleshooting.
- **Set up actionable alerts**: Alert on user-facing issues (errors, latency), not infrastructure issues like CPU (unless correlated with user impact).
- **Monitor key metrics**: Request rate, error rate, latency (RED method). For resources: CPU, memory, disk (USE method).
- **Use dashboards**: Create Grafana dashboards for quick health overview. Include SLI/SLO metrics.
- **Retention policies**: Balance storage costs with compliance needs. 30-90 days for logs, 1+ year for aggregated metrics.

---

## Question 3: How do you ensure zero-downtime deployment?

### Conceptual Explanation

Zero-downtime deployment ensures users experience no interruption during updates. Key strategies: **rolling updates** (gradually replace old pods), **blue-green deployment** (switch traffic between two environments), **canary deployments** (gradual traffic shift to new version), and **feature flags** (deploy code but enable features gradually).

Critical requirements: **backward-compatible changes** (new code works with old database schema), **graceful shutdown** (finish in-flight requests), **readiness probes** (don't route traffic until ready), and **health checks** (detect failed deployments quickly).

Database migrations need special care: use multi-phase migrations (add column → deploy code → remove old column), run migrations before deployment, and ensure they're backward compatible.

### Code Example

```yaml
# Rolling update deployment (Kubernetes default)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max 1 extra pod during update
      maxUnavailable: 1  # Max 1 pod down during update
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
      - name: myapp
        image: myapp:v2.0.0
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"] # Give load balancer time to deregister

---
# Blue-Green deployment with services
# Blue (current production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myapp:v1.0.0

---
# Green (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:v2.0.0

---
# Service points to blue initially
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: blue  # Switch to green after testing
  ports:
  - port: 80
    targetPort: 3000

# After validating green, switch service:
# kubectl patch service myapp-service -p '{"spec":{"selector":{"version":"green"}}}'
```

```javascript
// Feature flags for gradual rollout
class FeatureFlags {
  async isEnabled(userId, feature) {
    const rolloutPercentage = await this.getRolloutPercentage(feature);
    
    // Hash user ID for consistent experience
    const hash = this.hashString(userId);
    const bucket = hash % 100;
    
    return bucket < rolloutPercentage;
  }
  
  async getRolloutPercentage(feature) {
    // Fetch from config service or database
    const config = await redis.get(`feature:${feature}`);
    return config ? parseInt(config) : 0;
  }
  
  hashString(str) {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash |= 0;
    }
    return Math.abs(hash);
  }
}

const featureFlags = new FeatureFlags();

// Use in application
app.get('/api/products', async (req, res) => {
  const useNewAlgorithm = await featureFlags.isEnabled(req.user.id, 'new_recommendation_algo');
  
  let products;
  if (useNewAlgorithm) {
    products = await newRecommendationService.getProducts(req.user.id);
  } else {
    products = await recommendationService.getProducts(req.user.id);
  }
  
  res.json(products);
});

// Database migration strategy for zero downtime
// Phase 1: Add new column (backward compatible)
await db.query(`
  ALTER TABLE users 
  ADD COLUMN full_name VARCHAR(255)
`);

// Deploy code that writes to BOTH old and new columns
async function updateUser(userId, name) {
  await db.query(`
    UPDATE users 
    SET name = $1, full_name = $1 
    WHERE id = $2
  `, [name, userId]);
}

// Phase 2: Backfill existing data
await db.query(`
  UPDATE users 
  SET full_name = name 
  WHERE full_name IS NULL
`);

// Phase 3: Deploy code that reads from new column
async function getUser(userId) {
  const result = await db.query(`
    SELECT id, full_name as name, email 
    FROM users 
    WHERE id = $1
  `, [userId]);
  return result.rows[0];
}

// Phase 4: Remove old column (after all code updated)
await db.query(`ALTER TABLE users DROP COLUMN name`);
```

### Best Practices

- **Use rolling updates by default**: Kubernetes gradually replaces pods. Set appropriate maxSurge and maxUnavailable values.
- **Implement proper health checks**: Readiness probes prevent traffic to pods that aren't ready. Use real checks (DB connection, not just HTTP 200).
- **Graceful shutdown is critical**: Handle SIGTERM properly. Finish in-flight requests before exiting. Kubernetes waits 30s before SIGKILL.
- **Backward-compatible deployments**: New code must work with old database schema during rollout. Use multi-phase migrations.
- **Automated rollback**: Monitor error rates during deployment. Automatically rollback if thresholds exceeded.
- **Canary deployments for risk reduction**: Route small percentage of traffic to new version first. Gradually increase if metrics look good.
- **Test rollback procedures**: Practice rolling back in staging. Ensure rollback is as smooth as deployment.

---

## Question 4: What's your strategy for handling API versioning and backward compatibility?

### Conceptual Explanation

API versioning manages changes while maintaining compatibility with existing clients. Strategies include: **URL versioning** (`/v1/users`, `/v2/users`), **header versioning** (`Accept: application/vnd.api+json; version=2`), **query parameter** (`/users?version=2`), and **content negotiation**. URL versioning is most explicit and commonly used.

Backward compatibility principles: **additive changes are safe** (new fields, new endpoints), **breaking changes need new version** (removing fields, changing types, different validation), and **sunset policy** (deprecation timeline, typically 12-24 months).

For microservices, version at the service boundary. Internal services can evolve faster. Public APIs need stricter versioning. Use API gateways to handle routing and transformation between versions.

### Code Example

```javascript
// URL-based versioning
const express = require('express');
const app = express();

// V1 API
const v1Router = express.Router();

v1Router.get('/users/:id', async (req, res) => {
  const user = await db.query('SELECT id, name, email FROM users WHERE id = $1', [req.params.id]);
  res.json(user.rows[0]);
});

v1Router.post('/users', async (req, res) => {
  const { name, email } = req.body;
  const result = await db.query(
    'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
    [name, email]
  );
  res.status(201).json(result.rows[0]);
});

app.use('/api/v1', v1Router);

// V2 API with breaking changes
const v2Router = express.Router();

v2Router.get('/users/:id', async (req, res) => {
  const user = await db.query(`
    SELECT id, first_name, last_name, email, created_at 
    FROM users 
    WHERE id = $1
  `, [req.params.id]);
  
  // V2 returns different structure
  const userData = user.rows[0];
  res.json({
    id: userData.id,
    fullName: `${userData.first_name} ${userData.last_name}`,
    email: userData.email,
    profile: {
      createdAt: userData.created_at
    }
  });
});

v2Router.post('/users', async (req, res) => {
  const { firstName, lastName, email } = req.body; // Different field names
  
  // Validation changes in V2
  if (!firstName || !lastName || !email) {
    return res.status(400).json({ error: 'Missing required fields' });
  }
  
  const result = await db.query(
    'INSERT INTO users (first_name, last_name, email) VALUES ($1, $2, $3) RETURNING *',
    [firstName, lastName, email]
  );
  
  res.status(201).json({
    id: result.rows[0].id,
    fullName: `${firstName} ${lastName}`,
    email
  });
});

app.use('/api/v2', v2Router);

// Version detection with default
app.use('/api/users', (req, res) => {
  res.redirect('/api/v2/users'); // Redirect to latest version
});

// Header-based versioning alternative
app.use('/api', (req, res, next) => {
  const version = req.get('API-Version') || '1';
  req.apiVersion = version;
  next();
});

app.get('/api/users/:id', async (req, res) => {
  if (req.apiVersion === '2') {
    // V2 logic
    return v2GetUser(req, res);
  } else {
    // V1 logic (default)
    return v1GetUser(req, res);
  }
});

// Deprecation handling
const deprecatedEndpoints = {
  '/api/v1/users': {
    sunsetDate: '2025-12-31',
    replacement: '/api/v2/users',
    message: 'This endpoint will be removed on 2025-12-31. Please migrate to v2.'
  }
};

app.use((req, res, next) => {
  const deprecation = deprecatedEndpoints[req.path];
  
  if (deprecation) {
    res.set({
      'Deprecation': 'true',
      'Sunset': deprecation.sunsetDate,
      'Link': `<${deprecation.replacement}>; rel="successor-version"`
    });
    
    // Log for monitoring
    logger.warn({
      message: 'Deprecated endpoint used',
      path: req.path,
      client: req.get('User-Agent'),
      deprecation: deprecation.message
    });
  }
  
  next();
});

// Transforming between versions (adapter pattern)
class UserTransformer {
  toV1(user) {
    return {
      id: user.id,
      name: `${user.firstName} ${user.lastName}`,
      email: user.email
    };
  }
  
  toV2(user) {
    return {
      id: user.id,
      fullName: `${user.firstName} ${user.lastName}`,
      email: user.email,
      profile: {
        createdAt: user.createdAt,
        updatedAt: user.updatedAt
      }
    };
  }
  
  fromV1(data) {
    const [firstName, ...lastNameParts] = data.name.split(' ');
    return {
      firstName,
      lastName: lastNameParts.join(' '),
      email: data.email
    };
  }
  
  fromV2(data) {
    const [firstName, ...lastNameParts] = data.fullName.split(' ');
    return {
      firstName,
      lastName: lastNameParts.join(' '),
      email: data.email
    };
  }
}

// Shared service layer with version adapters
class UserService {
  async getUser(userId) {
    // Single source of truth
    const result = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
    return result.rows[0];
  }
}

// Version-specific controllers use transformers
const userService = new UserService();
const transformer = new UserTransformer();

v1Router.get('/users/:id', async (req, res) => {
  const user = await userService.getUser(req.params.id);
  res.json(transformer.toV1(user));
});

v2Router.get('/users/:id', async (req, res) => {
  const user = await userService.getUser(req.params.id);
  res.json(transformer.toV2(user));
});
```

```yaml
# API Gateway for version routing (Kong configuration)
services:
  - name: users-service-v1
    url: http://users-service:3000
    routes:
      - name: users-v1
        paths:
          - /api/v1/users
        strip_path: true

  - name: users-service-v2
    url: http://users-service:3000
    routes:
      - name: users-v2
        paths:
          - /api/v2/users
        strip_path: true

# Rate limiting per version
plugins:
  - name: rate-limiting
    service: users-service-v1
    config:
      minute: 100
      policy: local

  - name: rate-limiting
    service: users-service-v2
    config:
      minute: 1000  # Higher limit for newer version
      policy: local
```

### Best Practices

- **Use URL versioning for clarity**: `/api/v1/`, `/api/v2/` is explicit and easy to route. Most widely adopted pattern.
- **Version only when breaking changes**: Adding fields, new endpoints don't need new version. Removing/renaming fields or changing behavior does.
- **Keep versions minimal**: Supporting many versions is expensive. Aim for 2-3 active versions maximum.
- **Set sunset dates**: Clearly communicate deprecation timeline (12-24 months). Use Sunset HTTP header.
- **Make v1 backward compatible when possible**: Add optional fields instead of breaking changes. Use transformers to adapt internal changes.
- **Version at service boundaries**: Internal microservices can evolve freely. Version only public-facing APIs.
- **Document changes clearly**: Maintain changelog with migration guides. Highlight breaking vs non-breaking changes.
- **Monitor version usage**: Track which clients use which versions. Proactively reach out before sunsetting versions.

