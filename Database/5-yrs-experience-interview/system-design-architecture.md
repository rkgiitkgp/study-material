# System Design & Architecture - Interview Answers

## Question 1: How would you design a scalable notification or chat system using Node.js and Postgres?

### Conceptual Explanation

A scalable notification/chat system requires real-time message delivery, message persistence, user presence tracking, and horizontal scalability. The architecture typically involves WebSocket connections for real-time communication, a message broker for distributed delivery, PostgreSQL for persistent storage, and Redis for caching and pub/sub functionality.

The key components include: (1) WebSocket servers for maintaining persistent connections with clients, (2) Message queue (Kafka/RabbitMQ) for reliable message delivery across multiple server instances, (3) PostgreSQL for storing message history and user data with proper indexing, and (4) Redis for presence tracking, unread counts, and pub/sub patterns. The system should be stateless to allow horizontal scaling behind a load balancer.

For chat specifically, consider implementing concepts like conversation threads, read receipts, typing indicators, and message delivery acknowledgments. For notifications, implement priority queuing, batch processing for bulk notifications, and user preference management for notification channels (email, push, in-app).

### Code Example

```typescript
// WebSocket server with Socket.io
import { Server } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import Redis from 'ioredis';
import { Pool } from 'pg';

// Redis for pub/sub across multiple server instances
const pubClient = new Redis({ host: 'redis-host', port: 6379 });
const subClient = pubClient.duplicate();

const io = new Server(server, {
  cors: { origin: '*' }
});

// Enable Redis adapter for horizontal scaling
io.adapter(createAdapter(pubClient, subClient));

// PostgreSQL connection pool
const pool = new Pool({
  host: 'postgres-host',
  database: 'chat_db',
  max: 20,
  idleTimeoutMillis: 30000
});

// Handle socket connections
io.on('connection', async (socket) => {
  const userId = socket.handshake.auth.userId;
  
  // Join user to their room for targeted messages
  socket.join(`user:${userId}`);
  
  // Update user presence in Redis
  await pubClient.setex(`presence:${userId}`, 300, 'online');
  
  socket.on('send_message', async (data) => {
    const { conversationId, content, recipientId } = data;
    
    try {
      // Persist message to PostgreSQL
      const result = await pool.query(
        `INSERT INTO messages (conversation_id, sender_id, content, created_at)
         VALUES ($1, $2, $3, NOW())
         RETURNING id, created_at`,
        [conversationId, userId, content]
      );
      
      const message = {
        id: result.rows[0].id,
        conversationId,
        senderId: userId,
        content,
        createdAt: result.rows[0].created_at
      };
      
      // Emit to recipient in real-time (works across all server instances)
      io.to(`user:${recipientId}`).emit('new_message', message);
      
      // Increment unread count in Redis
      await pubClient.incr(`unread:${recipientId}:${conversationId}`);
      
      // Acknowledge to sender
      socket.emit('message_sent', { messageId: message.id });
      
    } catch (error) {
      socket.emit('error', { message: 'Failed to send message' });
    }
  });
  
  socket.on('disconnect', async () => {
    await pubClient.del(`presence:${userId}`);
  });
});

// Notification system with priority queue
import Bull from 'bull';

const notificationQueue = new Bull('notifications', {
  redis: { host: 'redis-host', port: 6379 }
});

// Add notification to queue
async function sendNotification(userId: string, type: string, data: any, priority: number = 5) {
  await notificationQueue.add(
    { userId, type, data },
    { priority, attempts: 3, backoff: { type: 'exponential', delay: 2000 } }
  );
}

// Process notifications
notificationQueue.process(async (job) => {
  const { userId, type, data } = job.data;
  
  // Store in database
  await pool.query(
    `INSERT INTO notifications (user_id, type, data, read, created_at)
     VALUES ($1, $2, $3, false, NOW())`,
    [userId, type, JSON.stringify(data)]
  );
  
  // Send real-time notification via WebSocket
  io.to(`user:${userId}`).emit('notification', { type, data });
  
  // Optionally send push notification or email based on user preferences
  const prefs = await getUserNotificationPreferences(userId);
  if (prefs.pushEnabled) {
    await sendPushNotification(userId, data);
  }
});
```

```sql
-- PostgreSQL schema for chat system
CREATE TABLE conversations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type VARCHAR(20) NOT NULL, -- 'direct' or 'group'
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE conversation_participants (
  conversation_id UUID REFERENCES conversations(id) ON DELETE CASCADE,
  user_id UUID NOT NULL,
  joined_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  last_read_at TIMESTAMP WITH TIME ZONE,
  PRIMARY KEY (conversation_id, user_id)
);

CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID REFERENCES conversations(id) ON DELETE CASCADE,
  sender_id UUID NOT NULL,
  content TEXT NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  deleted_at TIMESTAMP WITH TIME ZONE
);

-- Indexes for performance
CREATE INDEX idx_messages_conversation_created 
  ON messages(conversation_id, created_at DESC);
CREATE INDEX idx_messages_sender ON messages(sender_id);
CREATE INDEX idx_conversation_participants_user 
  ON conversation_participants(user_id);

-- Notifications table
CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL,
  type VARCHAR(50) NOT NULL,
  data JSONB NOT NULL,
  read BOOLEAN DEFAULT false,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_notifications_user_unread 
  ON notifications(user_id, created_at DESC) WHERE read = false;
CREATE INDEX idx_notifications_created 
  ON notifications(created_at) WHERE read = true; -- For cleanup
```

### Best Practices

- **Use Redis adapter for Socket.io**: Essential for horizontal scaling across multiple WebSocket servers. Without it, messages won't reach users connected to different server instances.
- **Implement connection pooling**: Use PostgreSQL connection pools (pg.Pool) with appropriate max connections to prevent database overload during high traffic.
- **Message queue for reliability**: Use Bull, BullMQ, or Kafka to ensure message delivery even if services restart. Implement retry logic with exponential backoff.
- **Optimize database queries**: Index on `(conversation_id, created_at)` for fast message retrieval. Use LIMIT and pagination for loading message history.
- **Presence management**: Store user presence in Redis with TTL (time-to-live) and refresh periodically. Use heartbeat mechanism to detect disconnections.
- **Rate limiting**: Implement rate limiting per user to prevent spam (e.g., max 10 messages per second using Redis counters).
- **Archive old data**: Move messages older than 90 days to archive tables or cold storage to keep main tables performant.

---

## Question 2: What factors do you consider before breaking a monolith into microservices?

### Conceptual Explanation

Breaking a monolith into microservices is a significant architectural decision that should be driven by clear business and technical needs, not trends. The key factors include team size and structure, bounded contexts within the domain, deployment independence requirements, scalability needs, and the organization's operational maturity.

Before proceeding, assess whether the monolith is truly limiting your team's velocity and scalability. Microservices introduce distributed system complexity: network failures, data consistency challenges, distributed tracing, service discovery, and increased operational overhead. The team must have strong DevOps capabilities, monitoring infrastructure, and understanding of distributed systems.

Consider the domain-driven design principle of bounded contexts. Each microservice should represent a clear business capability with well-defined boundaries. Look for areas with different scaling requirements, independent release cycles, or distinct team ownership. The Conway's Law principle suggests that system architecture will mirror organizational structure, so align service boundaries with team boundaries.

### Code Example

```typescript
// Before: Monolithic Express application
// app.ts - Everything in one codebase
import express from 'express';

const app = express();

// User management
app.post('/api/users', createUser);
app.get('/api/users/:id', getUser);

// Order processing
app.post('/api/orders', createOrder);
app.get('/api/orders/:id', getOrder);

// Payment processing
app.post('/api/payments', processPayment);

// Notification sending
app.post('/api/notifications/send', sendNotification);

// Inventory management
app.put('/api/inventory/:productId', updateInventory);

// All share same database connection
const db = createDbConnection();

// After: Microservices approach
// Service 1: User Service (users-service)
// Handles authentication, user profiles, permissions
import express from 'express';
const app = express();

app.post('/api/users', async (req, res) => {
  // Independent database
  const user = await userDb.create(req.body);
  
  // Emit event for other services
  await eventBus.publish('user.created', {
    userId: user.id,
    email: user.email,
    createdAt: user.createdAt
  });
  
  res.json(user);
});

// Service 2: Order Service (orders-service)
// Handles order lifecycle, order history
import express from 'express';
const app = express();

app.post('/api/orders', async (req, res) => {
  const { userId, items } = req.body;
  
  // Verify user exists (sync call or cached data)
  const userExists = await userServiceClient.checkUser(userId);
  if (!userExists) {
    return res.status(404).json({ error: 'User not found' });
  }
  
  // Check inventory (sync call)
  const inventoryAvailable = await inventoryServiceClient.checkAvailability(items);
  if (!inventoryAvailable) {
    return res.status(400).json({ error: 'Items not available' });
  }
  
  // Create order in own database
  const order = await orderDb.create({ userId, items, status: 'pending' });
  
  // Emit event for async processing
  await eventBus.publish('order.created', {
    orderId: order.id,
    userId,
    items,
    totalAmount: order.total
  });
  
  res.json(order);
});

// Service 3: Payment Service (payments-service)
// Listens to order events and processes payments
import { Kafka } from 'kafkajs';

const kafka = new Kafka({ brokers: ['kafka:9092'] });
const consumer = kafka.consumer({ groupId: 'payment-service' });

await consumer.subscribe({ topic: 'order.created' });

await consumer.run({
  eachMessage: async ({ message }) => {
    const order = JSON.parse(message.value.toString());
    
    try {
      // Process payment
      const payment = await paymentProcessor.charge(order);
      
      // Update own database
      await paymentDb.create({
        orderId: order.orderId,
        amount: order.totalAmount,
        status: 'completed'
      });
      
      // Emit success event
      await eventBus.publish('payment.completed', {
        orderId: order.orderId,
        paymentId: payment.id
      });
      
    } catch (error) {
      // Emit failure event
      await eventBus.publish('payment.failed', {
        orderId: order.orderId,
        reason: error.message
      });
    }
  }
});
```

```yaml
# Docker Compose for local development
version: '3.8'
services:
  user-service:
    build: ./services/user-service
    environment:
      DATABASE_URL: postgresql://user_db:5432/users
      KAFKA_BROKERS: kafka:9092
    ports:
      - "3001:3000"
  
  order-service:
    build: ./services/order-service
    environment:
      DATABASE_URL: postgresql://order_db:5432/orders
      KAFKA_BROKERS: kafka:9092
      USER_SERVICE_URL: http://user-service:3000
    ports:
      - "3002:3000"
  
  payment-service:
    build: ./services/payment-service
    environment:
      DATABASE_URL: postgresql://payment_db:5432/payments
      KAFKA_BROKERS: kafka:9092
    ports:
      - "3003:3000"
  
  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
```

### Best Practices

- **Start with the monolith**: Build a well-structured modular monolith first. Extract microservices only when you have clear pain points and sufficient operational maturity.
- **Identify bounded contexts**: Use domain-driven design to identify natural boundaries. Services should have high cohesion within and loose coupling between them.
- **Consider the data**: Can you separate the database? Shared databases defeat the purpose of microservices. Each service should own its data with APIs as the interface.
- **Evaluate operational readiness**: Do you have CI/CD pipelines, container orchestration (Kubernetes), centralized logging (ELK), distributed tracing (Jaeger), and on-call processes?
- **Communication patterns**: Choose synchronous (REST/gRPC) for immediate consistency needs and asynchronous (Kafka/RabbitMQ) for eventual consistency scenarios.
- **Team structure**: Ensure teams can independently develop, test, and deploy their services. Microservices with cross-team dependencies create bottlenecks.
- **Strangler pattern**: Gradually extract services from the monolith rather than a big-bang rewrite. Route traffic to new services while keeping the monolith as fallback.
- **Cost consideration**: Microservices increase infrastructure costs (more services, databases, monitoring tools) and development overhead. Ensure the benefits justify the costs.

---

## Question 3: How do you handle inter-service communication — REST vs. message queues (Kafka, RabbitMQ)?

### Conceptual Explanation

Inter-service communication falls into two categories: synchronous (REST, gRPC) and asynchronous (message queues). The choice depends on coupling requirements, consistency needs, latency tolerance, and failure handling strategies.

REST is ideal for request-response patterns where you need immediate results and can tolerate tight coupling. It's simple, widely understood, and works well with HTTP-based infrastructure (load balancers, API gateways). However, it creates cascading failures—if Service A calls Service B which calls Service C, a failure in C affects the entire chain. gRPC offers better performance with binary serialization and HTTP/2 but requires more setup.

Message queues (Kafka, RabbitMQ) enable asynchronous, event-driven communication. They decouple services—producers don't need to know about consumers. This provides better resilience (messages persist if a consumer is down), supports multiple consumers for the same event, and enables event sourcing patterns. The tradeoff is eventual consistency and increased complexity in debugging distributed flows.

### Code Example

```typescript
// Synchronous REST communication
// order-service calling inventory-service
import axios from 'axios';
import CircuitBreaker from 'opossum';

// Circuit breaker to prevent cascading failures
const inventoryClient = axios.create({
  baseURL: process.env.INVENTORY_SERVICE_URL,
  timeout: 3000 // 3 second timeout
});

const circuitBreakerOptions = {
  timeout: 3000,
  errorThresholdPercentage: 50,
  resetTimeout: 30000 // Try again after 30s
};

const checkInventoryBreaker = new CircuitBreaker(
  async (productId: string, quantity: number) => {
    const response = await inventoryClient.post('/check-availability', {
      productId,
      quantity
    });
    return response.data;
  },
  circuitBreakerOptions
);

// Order creation with synchronous inventory check
app.post('/api/orders', async (req, res) => {
  const { userId, productId, quantity } = req.body;
  
  try {
    // Synchronous call - blocks until response
    const available = await checkInventoryBreaker.fire(productId, quantity);
    
    if (!available) {
      return res.status(400).json({ error: 'Product not available' });
    }
    
    // Create order
    const order = await db.orders.create({
      userId,
      productId,
      quantity,
      status: 'pending'
    });
    
    // Reserve inventory (another sync call)
    await inventoryClient.post('/reserve', {
      orderId: order.id,
      productId,
      quantity
    });
    
    res.json(order);
    
  } catch (error) {
    if (error.message === 'Breaker is open') {
      // Fallback behavior when inventory service is down
      return res.status(503).json({ 
        error: 'Inventory service unavailable. Please try again later.' 
      });
    }
    res.status(500).json({ error: 'Order creation failed' });
  }
});

// Asynchronous event-driven communication with Kafka
import { Kafka } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'order-service',
  brokers: process.env.KAFKA_BROKERS.split(',')
});

const producer = kafka.producer();
const consumer = kafka.consumer({ groupId: 'order-service-group' });

await producer.connect();
await consumer.connect();

// Order creation with async processing
app.post('/api/orders', async (req, res) => {
  const { userId, productId, quantity } = req.body;
  
  // Create order immediately with 'processing' status
  const order = await db.orders.create({
    userId,
    productId,
    quantity,
    status: 'processing'
  });
  
  // Emit event for async processing - returns immediately
  await producer.send({
    topic: 'order.created',
    messages: [{
      key: order.id,
      value: JSON.stringify({
        orderId: order.id,
        userId,
        productId,
        quantity,
        timestamp: Date.now()
      })
    }]
  });
  
  // Return immediately without waiting for inventory
  res.status(202).json({ 
    orderId: order.id,
    status: 'processing',
    message: 'Order is being processed'
  });
});

// Inventory service consumes events asynchronously
await consumer.subscribe({ topic: 'order.created' });

await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    const order = JSON.parse(message.value.toString());
    
    try {
      // Check and reserve inventory
      const available = await checkAndReserveInventory(
        order.productId,
        order.quantity
      );
      
      if (available) {
        // Emit success event
        await producer.send({
          topic: 'inventory.reserved',
          messages: [{
            key: order.orderId,
            value: JSON.stringify({
              orderId: order.orderId,
              productId: order.productId,
              quantity: order.quantity
            })
          }]
        });
      } else {
        // Emit failure event
        await producer.send({
          topic: 'inventory.unavailable',
          messages: [{
            key: order.orderId,
            value: JSON.stringify({
              orderId: order.orderId,
              reason: 'Insufficient inventory'
            })
          }]
        });
      }
      
    } catch (error) {
      // Kafka will retry the message
      throw error;
    }
  }
});

// Order service listens for inventory events and updates order status
await consumer.subscribe({ topic: 'inventory.reserved' });
await consumer.subscribe({ topic: 'inventory.unavailable' });

await consumer.run({
  eachMessage: async ({ topic, message }) => {
    const data = JSON.parse(message.value.toString());
    
    if (topic === 'inventory.reserved') {
      await db.orders.update(
        { id: data.orderId },
        { status: 'confirmed' }
      );
      
      // Notify user
      await producer.send({
        topic: 'notification.send',
        messages: [{
          value: JSON.stringify({
            userId: data.userId,
            type: 'order_confirmed',
            orderId: data.orderId
          })
        }]
      });
      
    } else if (topic === 'inventory.unavailable') {
      await db.orders.update(
        { id: data.orderId },
        { status: 'cancelled', reason: data.reason }
      );
    }
  }
});
```

```typescript
// Hybrid approach - Saga pattern with compensation
// Uses events but maintains consistency through orchestration

class OrderSagaOrchestrator {
  async createOrder(orderData: OrderData) {
    const sagaId = generateId();
    
    try {
      // Step 1: Create order
      const order = await this.orderService.create(orderData);
      await this.saveSagaState(sagaId, 'order_created', { orderId: order.id });
      
      // Step 2: Reserve inventory (async via event, wait for response)
      const inventoryReserved = await this.reserveInventoryWithTimeout(
        order.productId,
        order.quantity,
        10000 // 10 second timeout
      );
      
      if (!inventoryReserved) {
        await this.compensate(sagaId, 'inventory_failed');
        throw new Error('Inventory reservation failed');
      }
      
      await this.saveSagaState(sagaId, 'inventory_reserved');
      
      // Step 3: Process payment
      const paymentSuccess = await this.processPaymentWithTimeout(
        order.id,
        order.totalAmount,
        30000 // 30 second timeout
      );
      
      if (!paymentSuccess) {
        await this.compensate(sagaId, 'payment_failed');
        throw new Error('Payment failed');
      }
      
      // Mark saga complete
      await this.saveSagaState(sagaId, 'completed');
      return order;
      
    } catch (error) {
      // Compensation already handled
      throw error;
    }
  }
  
  async compensate(sagaId: string, failurePoint: string) {
    const sagaState = await this.getSagaState(sagaId);
    
    // Rollback in reverse order
    if (sagaState.includes('inventory_reserved')) {
      await producer.send({
        topic: 'inventory.release',
        messages: [{ value: JSON.stringify({ sagaId }) }]
      });
    }
    
    if (sagaState.includes('order_created')) {
      await this.orderService.cancel(sagaState.orderId);
    }
  }
}
```

### Best Practices

- **Use REST for queries and immediate responses**: When you need data synchronously (e.g., user profile lookup, availability check), REST/gRPC is appropriate. Keep these calls lightweight and implement timeouts.
- **Use message queues for events and commands**: Order placement, notifications, analytics events, and long-running processes benefit from async messaging. This improves resilience and allows processing peaks.
- **Implement circuit breakers**: Prevent cascading failures in synchronous calls using patterns like Opossum or Hystrix. Open circuits when error rates exceed thresholds.
- **Consider consistency requirements**: Use synchronous calls when you need immediate consistency (strong consistency). Use async events for eventual consistency scenarios.
- **Idempotency is critical**: With message queues, messages may be delivered multiple times. Design consumers to handle duplicate messages safely using idempotency keys.
- **Monitor both patterns**: Track REST latencies, error rates, and circuit breaker states. Monitor message queue lag, processing rates, and dead letter queues.
- **Hybrid approaches work well**: Use Saga pattern or event-driven orchestration to combine benefits of both—async processing with compensating transactions for consistency.

---

## Question 4: How would you ensure data consistency between multiple microservices?

### Conceptual Explanation

Data consistency in microservices is challenging because each service owns its database, and traditional ACID transactions don't span multiple databases. You must choose between strong consistency (immediate, harder to achieve) and eventual consistency (delayed, more scalable).

The primary patterns include: (1) **Saga pattern** - a sequence of local transactions coordinated through events or orchestration, with compensating transactions for rollbacks; (2) **Event sourcing** - storing state changes as events, allowing rebuild of current state and maintaining audit trails; (3) **CQRS (Command Query Responsibility Segregation)** - separate models for writes and reads, with eventual synchronization; (4) **Two-Phase Commit (2PC)** - distributed transactions, but complex and can cause availability issues.

Most modern systems favor eventual consistency with the Saga pattern. Each service performs its local transaction and emits events. If a downstream service fails, compensating transactions rollback previous changes. This maintains consistency without distributed locks, though it requires careful design of compensation logic and idempotent operations.

### Code Example

```typescript
// Pattern 1: Choreography-based Saga (event-driven)
// Each service listens to events and emits new ones

// Order Service
class OrderService {
  async createOrder(orderData: CreateOrderDTO) {
    const order = await this.db.orders.create({
      ...orderData,
      status: 'pending',
      sagaState: 'order_created'
    });
    
    // Emit event for next step
    await this.eventBus.publish('order.created', {
      orderId: order.id,
      customerId: order.customerId,
      items: order.items,
      totalAmount: order.totalAmount
    });
    
    return order;
  }
  
  // Listen for payment success
  async onPaymentCompleted(event: PaymentCompletedEvent) {
    await this.db.orders.update(
      { id: event.orderId },
      { status: 'confirmed', sagaState: 'payment_completed' }
    );
    
    await this.eventBus.publish('order.confirmed', {
      orderId: event.orderId
    });
  }
  
  // Listen for payment failure - compensate
  async onPaymentFailed(event: PaymentFailedEvent) {
    await this.db.orders.update(
      { id: event.orderId },
      { status: 'cancelled', sagaState: 'payment_failed' }
    );
    
    // Trigger inventory release
    await this.eventBus.publish('inventory.release', {
      orderId: event.orderId
    });
  }
}

// Inventory Service
class InventoryService {
  async onOrderCreated(event: OrderCreatedEvent) {
    try {
      // Local transaction to reserve inventory
      await this.db.transaction(async (trx) => {
        for (const item of event.items) {
          const product = await trx('inventory')
            .where({ productId: item.productId })
            .forUpdate() // Lock row
            .first();
          
          if (product.quantity < item.quantity) {
            throw new Error('Insufficient inventory');
          }
          
          // Reserve inventory
          await trx('inventory')
            .where({ productId: item.productId })
            .decrement('quantity', item.quantity);
          
          // Track reservation
          await trx('reservations').insert({
            orderId: event.orderId,
            productId: item.productId,
            quantity: item.quantity,
            expiresAt: new Date(Date.now() + 15 * 60 * 1000) // 15 min
          });
        }
      });
      
      // Emit success event
      await this.eventBus.publish('inventory.reserved', {
        orderId: event.orderId,
        items: event.items
      });
      
    } catch (error) {
      // Emit failure event - triggers order cancellation
      await this.eventBus.publish('inventory.reservation.failed', {
        orderId: event.orderId,
        reason: error.message
      });
    }
  }
  
  // Compensating transaction
  async onInventoryRelease(event: InventoryReleaseEvent) {
    await this.db.transaction(async (trx) => {
      const reservations = await trx('reservations')
        .where({ orderId: event.orderId });
      
      for (const reservation of reservations) {
        await trx('inventory')
          .where({ productId: reservation.productId })
          .increment('quantity', reservation.quantity);
      }
      
      await trx('reservations')
        .where({ orderId: event.orderId })
        .delete();
    });
  }
}

// Payment Service
class PaymentService {
  async onInventoryReserved(event: InventoryReservedEvent) {
    try {
      // Process payment with external provider
      const paymentResult = await this.paymentProvider.charge({
        orderId: event.orderId,
        amount: event.totalAmount,
        idempotencyKey: `order-${event.orderId}` // Prevent double charge
      });
      
      await this.db.payments.create({
        orderId: event.orderId,
        amount: event.totalAmount,
        status: 'completed',
        providerId: paymentResult.id
      });
      
      await this.eventBus.publish('payment.completed', {
        orderId: event.orderId,
        paymentId: paymentResult.id
      });
      
    } catch (error) {
      await this.eventBus.publish('payment.failed', {
        orderId: event.orderId,
        reason: error.message
      });
    }
  }
}

// Pattern 2: Orchestration-based Saga
// Central orchestrator manages the saga flow

class OrderSagaOrchestrator {
  async executeOrderSaga(orderData: CreateOrderDTO): Promise<OrderResult> {
    const saga = await this.createSaga(orderData);
    
    try {
      // Step 1: Create order
      const order = await this.executeStep(saga, async () => {
        return await this.orderService.createOrder(orderData);
      }, async (order) => {
        // Compensation: cancel order
        await this.orderService.cancelOrder(order.id);
      });
      
      // Step 2: Reserve inventory
      await this.executeStep(saga, async () => {
        return await this.inventoryService.reserve(order.items);
      }, async () => {
        // Compensation: release inventory
        await this.inventoryService.release(order.id);
      });
      
      // Step 3: Process payment
      await this.executeStep(saga, async () => {
        return await this.paymentService.charge(order.id, order.total);
      }, async () => {
        // Compensation: refund payment
        await this.paymentService.refund(order.id);
      });
      
      // All steps successful
      await this.completeSaga(saga);
      return { success: true, orderId: order.id };
      
    } catch (error) {
      // Compensate all completed steps in reverse order
      await this.compensateSaga(saga);
      return { success: false, error: error.message };
    }
  }
  
  private async executeStep(
    saga: Saga,
    action: () => Promise<any>,
    compensation: (result: any) => Promise<void>
  ) {
    const stepId = generateId();
    
    try {
      const result = await action();
      
      // Save step state for potential compensation
      await this.saveSagaStep(saga.id, stepId, {
        status: 'completed',
        result,
        compensation: compensation.toString()
      });
      
      return result;
      
    } catch (error) {
      await this.saveSagaStep(saga.id, stepId, {
        status: 'failed',
        error: error.message
      });
      throw error;
    }
  }
  
  private async compensateSaga(saga: Saga) {
    const steps = await this.getSagaSteps(saga.id);
    
    // Execute compensations in reverse order
    for (const step of steps.reverse()) {
      if (step.status === 'completed') {
        try {
          await step.compensation(step.result);
          await this.updateStepStatus(step.id, 'compensated');
        } catch (error) {
          // Log compensation failure - may need manual intervention
          await this.logCompensationFailure(saga.id, step.id, error);
        }
      }
    }
  }
}

// Pattern 3: Outbox Pattern for reliable event publishing
// Ensures events are published exactly once

class OrderServiceWithOutbox {
  async createOrder(orderData: CreateOrderDTO) {
    // Single local transaction writes to both tables
    await this.db.transaction(async (trx) => {
      // Create order
      const order = await trx('orders').insert({
        ...orderData,
        status: 'pending'
      }).returning('*');
      
      // Write event to outbox table
      await trx('outbox_events').insert({
        aggregateType: 'order',
        aggregateId: order[0].id,
        eventType: 'order.created',
        payload: JSON.stringify({
          orderId: order[0].id,
          customerId: order[0].customerId,
          items: order[0].items
        }),
        createdAt: new Date()
      });
    });
  }
}

// Separate process publishes events from outbox
class OutboxPublisher {
  async publishPendingEvents() {
    const events = await this.db('outbox_events')
      .where('published', false)
      .orderBy('createdAt', 'asc')
      .limit(100);
    
    for (const event of events) {
      try {
        await this.eventBus.publish(event.eventType, JSON.parse(event.payload));
        
        await this.db('outbox_events')
          .where({ id: event.id })
          .update({ published: true, publishedAt: new Date() });
          
      } catch (error) {
        // Retry on next iteration
        console.error('Failed to publish event', event.id, error);
      }
    }
  }
  
  // Run every 1 second
  start() {
    setInterval(() => this.publishPendingEvents(), 1000);
  }
}
```

```sql
-- Schema for saga orchestration
CREATE TABLE sagas (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type VARCHAR(50) NOT NULL,
  status VARCHAR(20) NOT NULL, -- pending, completed, compensating, failed
  data JSONB NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE saga_steps (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  saga_id UUID REFERENCES sagas(id),
  step_name VARCHAR(100) NOT NULL,
  status VARCHAR(20) NOT NULL, -- completed, failed, compensated
  result JSONB,
  compensation_data JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Outbox pattern table
CREATE TABLE outbox_events (
  id BIGSERIAL PRIMARY KEY,
  aggregate_type VARCHAR(50) NOT NULL,
  aggregate_id UUID NOT NULL,
  event_type VARCHAR(100) NOT NULL,
  payload JSONB NOT NULL,
  published BOOLEAN DEFAULT false,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  published_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_outbox_unpublished ON outbox_events(created_at) 
  WHERE published = false;
```

### Best Practices

- **Favor eventual consistency**: Most business processes can tolerate slight delays. Design for eventual consistency using Saga pattern rather than distributed transactions.
- **Implement idempotency**: All operations must be idempotent since events may be delivered multiple times. Use idempotency keys (order IDs, event IDs) to detect duplicates.
- **Design compensating transactions carefully**: Not all operations can be reversed (e.g., sending email). Consider semantic compensation (send cancellation email) rather than technical rollback.
- **Use outbox pattern**: Ensure reliability by storing events in the same database transaction as business data, then publishing asynchronously.
- **Choose choreography for simple flows**: Event-driven choreography works well for 2-3 services. Use orchestration for complex flows with many steps and conditional logic.
- **Monitor saga states**: Track saga completion rates, average duration, and failure points. Alert on sagas stuck in intermediate states.
- **Set timeouts**: Implement timeouts for saga steps. After timeout, trigger compensation or manual intervention.
- **Version events**: Include version fields in events to handle schema evolution. Consumers should handle multiple event versions.

---

## Question 5: What caching strategy would you use in a high-read system (Redis, memory cache, etc.)?

### Conceptual Explanation

Caching strategies in high-read systems involve multiple layers: application-level in-memory cache (Node.js), distributed cache (Redis/Memcached), CDN caching (CloudFront/Cloudflare), and database query caching. The strategy depends on data access patterns, consistency requirements, memory constraints, and invalidation complexity.

Common patterns include: **Cache-Aside (Lazy Loading)** - application checks cache first, then database, and updates cache; **Write-Through** - write to cache and database simultaneously; **Write-Behind** - write to cache immediately, database asynchronously; **Read-Through** - cache automatically loads from database on miss. Each has tradeoffs between consistency, complexity, and performance.

For high-read systems, implement a multi-tier caching strategy: (1) In-memory cache (node-cache, lru-cache) for hot data with millisecond access, (2) Redis for distributed caching across instances with sub-10ms latency, (3) Database query cache for repeated queries, (4) HTTP cache headers for API responses. Use appropriate TTLs based on data volatility and implement smart invalidation to prevent stale data.

### Code Example

```typescript
// Multi-tier caching strategy
import NodeCache from 'node-cache';
import Redis from 'ioredis';
import { Pool } from 'pg';

// Tier 1: In-memory cache (fastest, limited to single instance)
const memoryCache = new NodeCache({
  stdTTL: 60, // 60 second default TTL
  checkperiod: 120, // Check for expired keys every 2 minutes
  useClones: false, // Better performance, but be careful with mutations
  maxKeys: 1000 // Limit memory usage
});

// Tier 2: Redis (distributed, shared across instances)
const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: 6379,
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,
  lazyConnect: false
});

// Database connection
const db = new Pool({ /* config */ });

// Cache-Aside pattern with multi-tier caching
class UserService {
  async getUserById(userId: string): Promise<User> {
    const cacheKey = `user:${userId}`;
    
    // Level 1: Check in-memory cache
    let user = memoryCache.get<User>(cacheKey);
    if (user) {
      console.log('Cache hit: memory');
      return user;
    }
    
    // Level 2: Check Redis
    const redisData = await redis.get(cacheKey);
    if (redisData) {
      console.log('Cache hit: Redis');
      user = JSON.parse(redisData);
      
      // Populate memory cache for next time
      memoryCache.set(cacheKey, user);
      return user;
    }
    
    // Level 3: Query database
    console.log('Cache miss: querying database');
    const result = await db.query(
      'SELECT * FROM users WHERE id = $1',
      [userId]
    );
    
    if (result.rows.length === 0) {
      throw new Error('User not found');
    }
    
    user = result.rows[0];
    
    // Populate both cache layers
    memoryCache.set(cacheKey, user);
    await redis.setex(cacheKey, 300, JSON.stringify(user)); // 5 min TTL
    
    return user;
  }
  
  // Write-Through pattern: update cache when data changes
  async updateUser(userId: string, updates: Partial<User>): Promise<User> {
    // Update database
    const result = await db.query(
      'UPDATE users SET name = $1, email = $2, updated_at = NOW() WHERE id = $3 RETURNING *',
      [updates.name, updates.email, userId]
    );
    
    const user = result.rows[0];
    const cacheKey = `user:${userId}`;
    
    // Update both cache layers immediately
    memoryCache.set(cacheKey, user);
    await redis.setex(cacheKey, 300, JSON.stringify(user));
    
    return user;
  }
  
  // Cache invalidation
  async deleteUser(userId: string): Promise<void> {
    await db.query('DELETE FROM users WHERE id = $1', [userId]);
    
    // Invalidate cache
    const cacheKey = `user:${userId}`;
    memoryCache.del(cacheKey);
    await redis.del(cacheKey);
  }
}

// Caching for computed/aggregated data
class AnalyticsService {
  async getProductStats(productId: string): Promise<ProductStats> {
    const cacheKey = `product:stats:${productId}`;
    
    // Try Redis first (no memory cache due to size)
    const cached = await redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }
    
    // Expensive aggregation query
    const result = await db.query(`
      SELECT 
        p.id,
        p.name,
        COUNT(DISTINCT o.id) as total_orders,
        SUM(oi.quantity) as total_sold,
        AVG(r.rating) as avg_rating,
        COUNT(DISTINCT r.id) as review_count
      FROM products p
      LEFT JOIN order_items oi ON p.id = oi.product_id
      LEFT JOIN orders o ON oi.order_id = o.id
      LEFT JOIN reviews r ON p.id = r.product_id
      WHERE p.id = $1
      GROUP BY p.id, p.name
    `, [productId]);
    
    const stats = result.rows[0];
    
    // Cache for 1 hour (stats don't need to be real-time)
    await redis.setex(cacheKey, 3600, JSON.stringify(stats));
    
    return stats;
  }
}

// Cache warming strategy
class CacheWarmer {
  async warmPopularProducts() {
    // Get top 100 products by views
    const result = await db.query(`
      SELECT id FROM products 
      ORDER BY view_count DESC 
      LIMIT 100
    `);
    
    const productIds = result.rows.map(row => row.id);
    
    // Warm cache in parallel
    await Promise.all(
      productIds.map(id => this.warmProductCache(id))
    );
    
    console.log(`Warmed cache for ${productIds.length} products`);
  }
  
  private async warmProductCache(productId: string) {
    const cacheKey = `product:${productId}`;
    
    const result = await db.query(
      'SELECT * FROM products WHERE id = $1',
      [productId]
    );
    
    if (result.rows.length > 0) {
      await redis.setex(cacheKey, 300, JSON.stringify(result.rows[0]));
    }
  }
  
  // Run every 5 minutes
  startWarming() {
    this.warmPopularProducts();
    setInterval(() => this.warmPopularProducts(), 5 * 60 * 1000);
  }
}

// HTTP caching with Express
import express from 'express';
const app = express();

app.get('/api/products/:id', async (req, res) => {
  const { id } = req.params;
  
  try {
    const product = await productService.getProduct(id);
    
    // Set HTTP cache headers
    res.set({
      'Cache-Control': 'public, max-age=300', // Cache for 5 minutes
      'ETag': `"${product.updatedAt.getTime()}"`, // Use timestamp as ETag
      'Last-Modified': product.updatedAt.toUTCString()
    });
    
    // Check if client has fresh cache
    const clientETag = req.headers['if-none-match'];
    if (clientETag === `"${product.updatedAt.getTime()}"`) {
      return res.status(304).end(); // Not Modified
    }
    
    res.json(product);
    
  } catch (error) {
    res.status(500).json({ error: 'Failed to fetch product' });
  }
});

// Cache invalidation using pub/sub
class CacheInvalidation {
  constructor(private redis: Redis) {
    this.setupSubscription();
  }
  
  private async setupSubscription() {
    const subscriber = this.redis.duplicate();
    
    await subscriber.subscribe('cache:invalidate');
    
    subscriber.on('message', (channel, message) => {
      const { pattern } = JSON.parse(message);
      
      // Invalidate matching keys
      this.invalidatePattern(pattern);
    });
  }
  
  async invalidatePattern(pattern: string) {
    // Invalidate memory cache
    const keys = memoryCache.keys();
    keys.forEach(key => {
      if (key.startsWith(pattern)) {
        memoryCache.del(key);
      }
    });
    
    // Invalidate Redis cache
    const redisKeys = await this.redis.keys(`${pattern}*`);
    if (redisKeys.length > 0) {
      await this.redis.del(...redisKeys);
    }
    
    console.log(`Invalidated cache pattern: ${pattern}`);
  }
  
  // Broadcast invalidation to all instances
  async broadcastInvalidation(pattern: string) {
    await this.redis.publish('cache:invalidate', JSON.stringify({ pattern }));
  }
}

// Usage: when product is updated
async function updateProduct(productId: string, data: any) {
  await db.query('UPDATE products SET ... WHERE id = $1', [productId]);
  
  // Invalidate all related caches
  await cacheInvalidation.broadcastInvalidation(`product:${productId}`);
  await cacheInvalidation.broadcastInvalidation(`product:stats:${productId}`);
}

// Advanced: Cache with stale-while-revalidate pattern
class StaleWhileRevalidateCache {
  async get(key: string, fetcher: () => Promise<any>, ttl: number) {
    const cacheKey = `swr:${key}`;
    const staleKey = `${cacheKey}:stale`;
    
    // Try to get fresh data
    const fresh = await redis.get(cacheKey);
    if (fresh) {
      return JSON.parse(fresh);
    }
    
    // Return stale data if available
    const stale = await redis.get(staleKey);
    if (stale) {
      // Trigger background refresh (don't await)
      this.refreshInBackground(key, cacheKey, staleKey, fetcher, ttl);
      return JSON.parse(stale);
    }
    
    // No cache at all, fetch synchronously
    const data = await fetcher();
    await this.setCache(cacheKey, staleKey, data, ttl);
    return data;
  }
  
  private async refreshInBackground(
    key: string,
    cacheKey: string,
    staleKey: string,
    fetcher: () => Promise<any>,
    ttl: number
  ) {
    try {
      const data = await fetcher();
      await this.setCache(cacheKey, staleKey, data, ttl);
    } catch (error) {
      console.error(`Failed to refresh cache for ${key}`, error);
    }
  }
  
  private async setCache(cacheKey: string, staleKey: string, data: any, ttl: number) {
    const serialized = JSON.stringify(data);
    
    // Set fresh cache with TTL
    await redis.setex(cacheKey, ttl, serialized);
    
    // Set stale cache with longer TTL
    await redis.setex(staleKey, ttl * 2, serialized);
  }
}
```

### Best Practices

- **Use layered caching**: Combine in-memory (fastest, single instance), Redis (fast, distributed), and HTTP caching (browser/CDN). Each layer serves different needs.
- **Set appropriate TTLs**: Short TTL (30-60s) for frequently changing data, longer TTL (5-30min) for stable data. Use cache warming for critical data on startup.
- **Implement cache invalidation**: Prefer explicit invalidation over relying only on TTL. Use pub/sub to broadcast invalidations across instances.
- **Monitor cache hit rates**: Track hit/miss ratios for each cache layer. Low hit rates indicate inefficient caching or incorrect TTLs. Aim for >80% hit rate.
- **Handle cache failures gracefully**: Redis downtime shouldn't bring down your app. Implement try-catch blocks and fallback to database queries.
- **Use cache keys wisely**: Include version numbers in keys for schema changes (`user:v2:${id}`). Use consistent naming patterns for easier invalidation.
- **Consider memory limits**: In-memory caches can cause OOM errors. Use LRU eviction and set maxKeys limits. Monitor Node.js heap usage.
- **Avoid caching everything**: Cache only frequently accessed or expensive-to-compute data. Don't cache user-specific data that's rarely reused.

---

## Question 6: Explain how you'd design a rate limiter or job scheduling system in Node.js.

### Conceptual Explanation

A rate limiter controls the frequency of operations to prevent abuse, ensure fair resource distribution, and protect downstream services. Common algorithms include: **Token Bucket** (flexible burst handling), **Leaky Bucket** (smooth rate), **Fixed Window** (simple but allows burst at boundaries), and **Sliding Window** (accurate but complex). Implementation requires distributed state (Redis) for multi-instance deployments.

Job scheduling systems manage background tasks, delayed execution, and recurring jobs. Key requirements include: job persistence (survive restarts), retry logic with backoff, priority queuing, concurrency control, and failure handling. Popular Node.js solutions include Bull/BullMQ (Redis-based), Agenda (MongoDB-based), and node-cron (simple time-based).

For enterprise systems, consider: idempotency (jobs may run twice), monitoring and alerting, job timeout handling, dead letter queues for failed jobs, and graceful shutdown to complete in-progress jobs. Rate limiting and job scheduling often work together—rate limiting prevents job queue overload while scheduling ensures orderly processing.

### Code Example

```typescript
// Rate Limiter - Token Bucket Algorithm with Redis
import Redis from 'ioredis';
import { Request, Response, NextFunction } from 'express';

class TokenBucketRateLimiter {
  constructor(
    private redis: Redis,
    private maxTokens: number = 10,
    private refillRate: number = 10, // tokens per second
    private refillInterval: number = 1000 // 1 second
  ) {}
  
  async checkLimit(identifier: string): Promise<{ allowed: boolean; remaining: number }> {
    const key = `ratelimit:${identifier}`;
    const now = Date.now();
    
    // Lua script for atomic token bucket operation
    const script = `
      local key = KEYS[1]
      local max_tokens = tonumber(ARGV[1])
      local refill_rate = tonumber(ARGV[2])
      local refill_interval = tonumber(ARGV[3])
      local now = tonumber(ARGV[4])
      
      local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
      local tokens = tonumber(bucket[1])
      local last_refill = tonumber(bucket[2])
      
      if tokens == nil then
        tokens = max_tokens
        last_refill = now
      end
      
      -- Calculate tokens to add based on time elapsed
      local time_passed = now - last_refill
      local tokens_to_add = math.floor((time_passed / refill_interval) * refill_rate)
      tokens = math.min(max_tokens, tokens + tokens_to_add)
      
      if tokens_to_add > 0 then
        last_refill = now
      end
      
      local allowed = 0
      if tokens >= 1 then
        tokens = tokens - 1
        allowed = 1
      end
      
      redis.call('HMSET', key, 'tokens', tokens, 'last_refill', last_refill)
      redis.call('EXPIRE', key, 3600)
      
      return {allowed, tokens}
    `;
    
    const result = await this.redis.eval(
      script,
      1,
      key,
      this.maxTokens,
      this.refillRate,
      this.refillInterval,
      now
    ) as [number, number];
    
    return {
      allowed: result[0] === 1,
      remaining: result[1]
    };
  }
}

// Express middleware
const rateLimiter = new TokenBucketRateLimiter(redis, 100, 10, 1000);

async function rateLimitMiddleware(req: Request, res: Response, next: NextFunction) {
  // Use user ID or IP address as identifier
  const identifier = req.user?.id || req.ip;
  
  const result = await rateLimiter.checkLimit(identifier);
  
  // Set rate limit headers
  res.setHeader('X-RateLimit-Remaining', result.remaining.toString());
  res.setHeader('X-RateLimit-Reset', Date.now() + 1000);
  
  if (!result.allowed) {
    return res.status(429).json({
      error: 'Too many requests',
      retryAfter: 1 // seconds
    });
  }
  
  next();
}

// Apply to routes
app.post('/api/orders', rateLimitMiddleware, createOrder);

// Sliding Window Rate Limiter (more accurate)
class SlidingWindowRateLimiter {
  constructor(
    private redis: Redis,
    private maxRequests: number = 100,
    private windowMs: number = 60000 // 1 minute
  ) {}
  
  async checkLimit(identifier: string): Promise<boolean> {
    const key = `ratelimit:sliding:${identifier}`;
    const now = Date.now();
    const windowStart = now - this.windowMs;
    
    // Lua script for atomic sliding window
    const script = `
      local key = KEYS[1]
      local window_start = tonumber(ARGV[1])
      local now = tonumber(ARGV[2])
      local max_requests = tonumber(ARGV[3])
      local window_ms = tonumber(ARGV[4])
      
      -- Remove old entries
      redis.call('ZREMRANGEBYSCORE', key, 0, window_start)
      
      -- Count requests in current window
      local count = redis.call('ZCARD', key)
      
      if count < max_requests then
        redis.call('ZADD', key, now, now)
        redis.call('EXPIRE', key, math.ceil(window_ms / 1000))
        return 1
      end
      
      return 0
    `;
    
    const allowed = await this.redis.eval(
      script,
      1,
      key,
      windowStart,
      now,
      this.maxRequests,
      this.windowMs
    ) as number;
    
    return allowed === 1;
  }
}

// Job Scheduling System using Bull
import Bull, { Queue, Job } from 'bull';

// Define job types
interface EmailJobData {
  to: string;
  subject: string;
  body: string;
}

interface ReportJobData {
  userId: string;
  reportType: string;
  dateRange: { start: Date; end: Date };
}

// Create queues
const emailQueue: Queue<EmailJobData> = new Bull('email', {
  redis: {
    host: process.env.REDIS_HOST,
    port: 6379
  },
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 2000
    },
    removeOnComplete: 100, // Keep last 100 completed jobs
    removeOnFail: 1000 // Keep last 1000 failed jobs
  }
});

const reportQueue: Queue<ReportJobData> = new Bull('reports', {
  redis: { host: process.env.REDIS_HOST },
  defaultJobOptions: {
    attempts: 2,
    timeout: 300000 // 5 minutes
  }
});

// Process jobs
emailQueue.process(async (job: Job<EmailJobData>) => {
  console.log(`Processing email job ${job.id}`);
  
  const { to, subject, body } = job.data;
  
  // Update progress
  await job.progress(10);
  
  // Send email via SMTP/SendGrid/etc
  await emailService.send({ to, subject, body });
  
  await job.progress(100);
  
  return { sent: true, timestamp: new Date() };
});

// Process with concurrency control
reportQueue.process(5, async (job: Job<ReportJobData>) => {
  console.log(`Generating report ${job.id} for user ${job.data.userId}`);
  
  const { userId, reportType, dateRange } = job.data;
  
  await job.progress(20);
  
  // Fetch data
  const data = await fetchReportData(userId, reportType, dateRange);
  await job.progress(60);
  
  // Generate PDF
  const pdfBuffer = await generatePDF(data);
  await job.progress(80);
  
  // Upload to S3
  const url = await uploadToS3(pdfBuffer, `reports/${userId}/${job.id}.pdf`);
  await job.progress(100);
  
  // Notify user
  await emailQueue.add({
    to: data.userEmail,
    subject: 'Your report is ready',
    body: `Download: ${url}`
  });
  
  return { url, generatedAt: new Date() };
});

// Add jobs to queue
async function sendWelcomeEmail(userEmail: string) {
  await emailQueue.add(
    {
      to: userEmail,
      subject: 'Welcome!',
      body: 'Thanks for signing up'
    },
    {
      priority: 1, // Higher priority
      delay: 5000 // Send after 5 seconds
    }
  );
}

async function generateMonthlyReport(userId: string) {
  await reportQueue.add(
    {
      userId,
      reportType: 'monthly',
      dateRange: {
        start: new Date('2024-01-01'),
        end: new Date('2024-01-31')
      }
    },
    {
      jobId: `monthly-report-${userId}-202401`, // Idempotent job ID
      priority: 3
    }
  );
}

// Scheduled recurring jobs
import cron from 'node-cron';

// Run every day at 2 AM
cron.schedule('0 2 * * *', async () => {
  console.log('Running daily cleanup job');
  
  await reportQueue.add({
    type: 'cleanup',
    olderThan: 90 // days
  });
});

// Run every hour
cron.schedule('0 * * * *', async () => {
  console.log('Running hourly sync job');
  
  await emailQueue.add({
    type: 'sync',
    timestamp: new Date()
  });
});

// Job event listeners
emailQueue.on('completed', (job, result) => {
  console.log(`Job ${job.id} completed:`, result);
});

emailQueue.on('failed', (job, err) => {
  console.error(`Job ${job.id} failed:`, err.message);
  
  // Alert if job failed after all retries
  if (job.attemptsMade >= job.opts.attempts) {
    alerting.send({
      level: 'error',
      message: `Email job ${job.id} failed permanently`,
      details: { job: job.data, error: err.message }
    });
  }
});

emailQueue.on('stalled', (job) => {
  console.warn(`Job ${job.id} stalled (possibly crashed worker)`);
});

// Graceful shutdown
async function gracefulShutdown() {
  console.log('Closing queues gracefully...');
  
  // Wait for active jobs to complete
  await emailQueue.close();
  await reportQueue.close();
  
  console.log('All queues closed');
  process.exit(0);
}

process.on('SIGTERM', gracefulShutdown);
process.on('SIGINT', gracefulShutdown);

// Job monitoring and metrics
class JobMonitor {
  async getQueueMetrics(queue: Queue) {
    const [waiting, active, completed, failed, delayed] = await Promise.all([
      queue.getWaitingCount(),
      queue.getActiveCount(),
      queue.getCompletedCount(),
      queue.getFailedCount(),
      queue.getDelayedCount()
    ]);
    
    return {
      waiting,
      active,
      completed,
      failed,
      delayed,
      total: waiting + active + completed + failed + delayed
    };
  }
  
  async getJobProcessingTime(queue: Queue, limit: number = 100) {
    const completed = await queue.getCompleted(0, limit - 1);
    
    const processingTimes = completed.map(job => {
      return job.finishedOn - job.processedOn;
    });
    
    const avg = processingTimes.reduce((a, b) => a + b, 0) / processingTimes.length;
    const max = Math.max(...processingTimes);
    const min = Math.min(...processingTimes);
    
    return { avg, max, min };
  }
}

// Expose metrics endpoint
app.get('/metrics/queues', async (req, res) => {
  const monitor = new JobMonitor();
  
  const metrics = {
    email: await monitor.getQueueMetrics(emailQueue),
    reports: await monitor.getQueueMetrics(reportQueue)
  };
  
  res.json(metrics);
});
```

### Best Practices

- **Use Redis for distributed rate limiting**: In-memory rate limiting doesn't work across multiple Node.js instances. Redis ensures consistent limits across all servers.
- **Implement multiple rate limit tiers**: Different limits for authenticated users, API keys, and anonymous requests. Consider per-endpoint limits for expensive operations.
- **Use Lua scripts for atomicity**: Redis Lua scripts execute atomically, preventing race conditions in rate limit checks. Critical for accurate counting.
- **Choose the right algorithm**: Token bucket for burst handling, sliding window for strict limits. Fixed window is simplest but allows double the limit at window boundaries.
- **Job idempotency is essential**: Use unique job IDs to prevent duplicate processing. Check if job already completed before expensive operations.
- **Implement job timeouts**: Prevent stuck jobs from blocking queues. Set realistic timeouts based on expected processing time.
- **Monitor queue health**: Track queue depth, processing rates, and failure rates. Alert on growing backlogs or high failure rates.
- **Graceful shutdown**: Wait for active jobs to complete before shutting down workers. Prevents data loss and incomplete operations.
- **Use priority queues wisely**: Critical jobs (password reset) get higher priority than bulk operations (daily reports).

---

## Question 7: How would you scale Postgres horizontally for millions of users?

### Conceptual Explanation

PostgreSQL is primarily designed for vertical scaling, but horizontal scaling is achievable through read replicas, sharding, connection pooling, and caching strategies. The approach depends on whether you're read-heavy, write-heavy, or both.

For read-heavy workloads, use **read replicas** (streaming replication) to distribute SELECT queries across multiple database instances. For write-heavy workloads, implement **sharding** (partitioning data across multiple databases) based on user ID, geographic region, or tenant ID. This requires application-level routing logic to direct queries to the correct shard.

Connection pooling is critical at scale—each PostgreSQL connection consumes significant memory (typically 10MB). Use PgBouncer or connection poolers to maintain a small number of database connections while serving thousands of application connections. Combine with caching (Redis) to reduce database load, and implement database-level optimizations (indexing, partitioning, vacuuming) before considering horizontal scaling.

### Code Example

```typescript
// Strategy 1: Read Replicas with pg-pool for load distribution
import { Pool } from 'pg';

class PostgresConnectionManager {
  private primaryPool: Pool;
  private replicaPools: Pool[];
  private currentReplicaIndex: number = 0;
  
  constructor() {
    // Primary database - handles all writes
    this.primaryPool = new Pool({
      host: process.env.DB_PRIMARY_HOST,
      port: 5432,
      database: 'myapp',
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      max: 20, // Maximum connections
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 5000
    });
    
    // Read replicas - handle read queries
    this.replicaPools = [
      new Pool({
        host: process.env.DB_REPLICA1_HOST,
        port: 5432,
        database: 'myapp',
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        max: 20
      }),
      new Pool({
        host: process.env.DB_REPLICA2_HOST,
        port: 5432,
        database: 'myapp',
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        max: 20
      })
    ];
  }
  
  // Round-robin load balancing across replicas
  getReplicaPool(): Pool {
    const pool = this.replicaPools[this.currentReplicaIndex];
    this.currentReplicaIndex = (this.currentReplicaIndex + 1) % this.replicaPools.length;
    return pool;
  }
  
  getPrimaryPool(): Pool {
    return this.primaryPool;
  }
  
  // Execute read query on replica
  async query(sql: string, params?: any[]) {
    const pool = this.getReplicaPool();
    return pool.query(sql, params);
  }
  
  // Execute write query on primary
  async execute(sql: string, params?: any[]) {
    return this.primaryPool.query(sql, params);
  }
  
  // Transaction always on primary
  async transaction<T>(callback: (client: any) => Promise<T>): Promise<T> {
    const client = await this.primaryPool.connect();
    
    try {
      await client.query('BEGIN');
      const result = await callback(client);
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
}

const db = new PostgresConnectionManager();

// Usage in application
class UserService {
  // Read from replica
  async getUserById(userId: string): Promise<User> {
    const result = await db.query(
      'SELECT * FROM users WHERE id = $1',
      [userId]
    );
    return result.rows[0];
  }
  
  // Write to primary
  async createUser(userData: CreateUserDTO): Promise<User> {
    const result = await db.execute(
      'INSERT INTO users (name, email, created_at) VALUES ($1, $2, NOW()) RETURNING *',
      [userData.name, userData.email]
    );
    return result.rows[0];
  }
  
  // Transaction on primary
  async transferCredits(fromUserId: string, toUserId: string, amount: number) {
    return db.transaction(async (client) => {
      // Deduct from sender
      await client.query(
        'UPDATE users SET credits = credits - $1 WHERE id = $2 AND credits >= $1',
        [amount, fromUserId]
      );
      
      // Add to receiver
      await client.query(
        'UPDATE users SET credits = credits + $1 WHERE id = $2',
        [amount, toUserId]
      );
      
      // Log transaction
      await client.query(
        'INSERT INTO transactions (from_user, to_user, amount) VALUES ($1, $2, $3)',
        [fromUserId, toUserId, amount]
      );
    });
  }
}

// Strategy 2: Sharding by user ID
class ShardedPostgresManager {
  private shards: Map<number, Pool>;
  private numShards: number;
  
  constructor(shardConfigs: Array<{ host: string; port: number }>) {
    this.numShards = shardConfigs.length;
    this.shards = new Map();
    
    shardConfigs.forEach((config, index) => {
      this.shards.set(index, new Pool({
        host: config.host,
        port: config.port,
        database: 'myapp',
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        max: 20
      }));
    });
  }
  
  // Consistent hashing to determine shard
  getShardForUser(userId: string): Pool {
    // Simple hash function (use better hash in production)
    const hash = userId.split('').reduce((acc, char) => {
      return ((acc << 5) - acc) + char.charCodeAt(0);
    }, 0);
    
    const shardId = Math.abs(hash) % this.numShards;
    return this.shards.get(shardId)!;
  }
  
  // Query single shard
  async queryUserShard(userId: string, sql: string, params: any[]) {
    const shard = this.getShardForUser(userId);
    return shard.query(sql, params);
  }
  
  // Query all shards (expensive!)
  async queryAllShards(sql: string, params: any[]) {
    const promises = Array.from(this.shards.values()).map(shard =>
      shard.query(sql, params)
    );
    
    const results = await Promise.all(promises);
    
    // Merge results from all shards
    return {
      rows: results.flatMap(r => r.rows),
      rowCount: results.reduce((sum, r) => sum + r.rowCount, 0)
    };
  }
}

const shardedDb = new ShardedPostgresManager([
  { host: 'shard1.db.com', port: 5432 },
  { host: 'shard2.db.com', port: 5432 },
  { host: 'shard3.db.com', port: 5432 },
  { host: 'shard4.db.com', port: 5432 }
]);

// Usage with sharding
class ShardedUserService {
  async getUserById(userId: string): Promise<User> {
    const result = await shardedDb.queryUserShard(
      userId,
      'SELECT * FROM users WHERE id = $1',
      [userId]
    );
    return result.rows[0];
  }
  
  async getUserOrders(userId: string): Promise<Order[]> {
    // Query the correct shard
    const result = await shardedDb.queryUserShard(
      userId,
      'SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at DESC',
      [userId]
    );
    return result.rows;
  }
  
  // Cross-shard query (avoid if possible!)
  async getTotalUserCount(): Promise<number> {
    const result = await shardedDb.queryAllShards(
      'SELECT COUNT(*) as count FROM users',
      []
    );
    return result.rows.reduce((sum, row) => sum + parseInt(row.count), 0);
  }
}

// Strategy 3: Connection pooling with PgBouncer
// PgBouncer configuration (pgbouncer.ini)
/*
[databases]
myapp = host=postgres-primary port=5432 dbname=myapp

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 10000
default_pool_size = 25
reserve_pool_size = 5
reserve_pool_timeout = 3
*/

// Application connects to PgBouncer instead of PostgreSQL
const pooledDb = new Pool({
  host: process.env.PGBOUNCER_HOST,
  port: 6432, // PgBouncer port
  database: 'myapp',
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 100 // Application pool, PgBouncer limits actual DB connections
});

// Strategy 4: Table partitioning for large tables
/*
-- Partition users table by created_at (time-based partitioning)
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(255),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Create partitions
CREATE TABLE users_2024_01 PARTITION OF users
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE users_2024_02 PARTITION OF users
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Indexes on each partition
CREATE INDEX idx_users_2024_01_email ON users_2024_01(email);
CREATE INDEX idx_users_2024_02_email ON users_2024_02(email);

-- Hash partitioning by user ID for even distribution
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL,
  total DECIMAL(10,2),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
) PARTITION BY HASH (user_id);

CREATE TABLE orders_p0 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE orders_p1 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE orders_p2 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE orders_p3 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 3);
*/

// Strategy 5: Citus for distributed PostgreSQL
// Citus extends PostgreSQL with distributed table functionality

/*
-- Enable Citus extension
CREATE EXTENSION citus;

-- Add worker nodes
SELECT * from citus_add_node('worker-1', 5432);
SELECT * from citus_add_node('worker-2', 5432);
SELECT * from citus_add_node('worker-3', 5432);

-- Create distributed table
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email VARCHAR(255),
  name VARCHAR(255),
  created_at TIMESTAMP WITH TIME ZONE
);

SELECT create_distributed_table('users', 'id');

-- Create reference table (replicated on all nodes)
CREATE TABLE countries (
  code VARCHAR(2) PRIMARY KEY,
  name VARCHAR(255)
);

SELECT create_reference_table('countries');

-- Queries work transparently across shards
INSERT INTO users (id, email, name) VALUES (gen_random_uuid(), 'user@example.com', 'John');
SELECT * FROM users WHERE email = 'user@example.com';
*/
```

```yaml
# Docker Compose for PostgreSQL with replicas
version: '3.8'
services:
  postgres-primary:
    image: postgres:15
    environment:
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    command: |
      postgres
      -c wal_level=replica
      -c max_wal_senders=3
      -c max_replication_slots=3
    volumes:
      - primary_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
  
  postgres-replica1:
    image: postgres:15
    environment:
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: secret
      PGUSER: replicator
      PGPASSWORD: replicator_password
    command: |
      bash -c "
      until pg_basebackup --pgdata=/var/lib/postgresql/data -R --slot=replication_slot_1 --host=postgres-primary --port=5432
      do
        echo 'Waiting for primary to connect...'
        sleep 1s
      done
      echo 'Backup done, starting replica...'
      postgres
      "
    depends_on:
      - postgres-primary
    ports:
      - "5433:5432"
  
  postgres-replica2:
    image: postgres:15
    environment:
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: secret
      PGUSER: replicator
      PGPASSWORD: replicator_password
    command: |
      bash -c "
      until pg_basebackup --pgdata=/var/lib/postgresql/data -R --slot=replication_slot_2 --host=postgres-primary --port=5432
      do
        echo 'Waiting for primary to connect...'
        sleep 1s
      done
      echo 'Backup done, starting replica...'
      postgres
      "
    depends_on:
      - postgres-primary
    ports:
      - "5434:5432"
  
  pgbouncer:
    image: edoburu/pgbouncer:latest
    environment:
      DATABASE_URL: postgresql://myapp:secret@postgres-primary:5432/myapp
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 10000
      DEFAULT_POOL_SIZE: 25
    ports:
      - "6432:5432"
    depends_on:
      - postgres-primary

volumes:
  primary_data:
```

### Best Practices

- **Start with read replicas**: Most applications are read-heavy. Streaming replication is built into PostgreSQL and requires minimal application changes. Route SELECT queries to replicas.
- **Use PgBouncer for connection pooling**: Essential for high-traffic apps. Allows thousands of app connections with only 20-50 actual database connections. Use transaction pooling mode.
- **Shard carefully**: Sharding adds significant complexity. Choose shard keys that align with access patterns (user ID, tenant ID). Avoid cross-shard queries in hot paths.
- **Optimize before scaling**: Add appropriate indexes, use EXPLAIN ANALYZE, optimize slow queries, implement table partitioning, and tune PostgreSQL settings (work_mem, shared_buffers).
- **Monitor replication lag**: Replicas may lag behind primary during high write load. Monitor lag and route time-sensitive reads to primary if needed.
- **Consider managed solutions**: Amazon RDS, Google Cloud SQL, and Azure Database offer read replicas, automated backups, and high availability without operational overhead.
- **Cache aggressively**: Use Redis to cache frequently accessed data. This often eliminates the need for complex scaling strategies.
- **Implement circuit breakers**: Protect your app from database outages. Degrade gracefully when databases are unavailable.

