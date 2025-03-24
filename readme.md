# WhatsApp Integration Platform: Master-Worker Architecture

## 1. Executive Overview

This architecture defines a scalable, resilient WhatsApp integration platform capable of managing multiple WhatsApp sessions concurrently while maintaining performance under high loads. The design addresses WhatsApp Web's fundamental limitation requiring single authentication instance per session through a robust Master-Worker pattern with dynamic scaling capabilities.

The platform enables:
- Parallel processing of resource-intensive operations
- Dynamic scaling based on workload
- Resilience against component failures
- Secure session management across distributed components

## 2. System Components

### 2.1 Master Process
**Responsibilities:**
- Client API request handling
- WhatsApp session lifecycle management
- Scheduling and orchestration of background tasks
- Task distribution via message queue
- System health monitoring

**Technology:** Node.js, Express.js

### 2.2 Worker Processes
**Responsibilities:**
- Execution of resource-intensive operations
- Processing of broadcast messages
- Group management operations
- Data scraping and analytics tasks
- Media processing

**Technology:** Node.js with WhatsApp Web.js

### 2.3 Instance Manager
**Responsibilities:**
- Worker process scaling based on demand metrics
- WhatsApp session initialization in worker processes
- Session-to-worker routing and affinity
- Worker health monitoring and recovery

**Technology:** Node.js, Kubernetes (recommended)

### 2.4 Message Queue
**Responsibilities:**
- Reliable asynchronous communication
- Task persistence with delivery guarantees
- Load balancing across available workers
- Priority-based task scheduling
+---------------------+      +---------------------+      +---------------------+
                            ^                               +---------------------+
                            |                               | * API Handling      |
                            |                               | * Session Mgmt      |
                            |                               | * Enqueue Tasks     |
                            |                               +---------------------+
                            |                                       |
                            |                                       V
                            |      +---------------------+      +---------------------+
                            ------->|     Message Queue   |----->|    Worker Process   |
                                   |     (RabbitMQ/Kafka)|      +---------------------+
                                   +---------------------+      | * Task Execution    |
                                                                | * WhatsApp Actions  |
                                                                +---------------------+
                                                                       ^
                                                                       |
                                     +---------------------+      +---------------------+
                                     |   Instance Manager  |----->|   Redis (Auth Data)|
                                     +---------------------+      +---------------------+
                                     | * Scale Workers     |
                                     | * Session Init      |
                                     +---------------------+

## Key Design Decisions

*   **Master-Worker Pattern:** Decouples API handling and session management from resource-intensive tasks, improving scalability and responsiveness.
*   **Instance Manager:** Dynamically scales worker processes based on workload, optimizing resource utilization.
*   **Message Queue:** Enables asynchronous communication and task persistence.
*   **Redis for Authentication Data:** Provides fast and secure access to authentication data for worker processes.
*   **Session Affinity:** Ensures that tasks related to a specific WhatsApp session are routed to the same worker process.
*   **Kubernetes (Recommended):** Simplifies the deployment, scaling, and management of worker processes.

## Scaling Strategy

### Worker VM Scaling Dimensions

Worker processes should be scaled according to these guidelines for optimal performance:

* **Memory-based allocation**: 4-8 WhatsApp sessions per 4GB RAM (each session uses ~150-300MB)
* **CPU-based allocation**: 4-8 sessions per 2 vCPUs
* **Activity-based**: Fewer sessions for high-message-volume accounts
* **Session type consideration**: Schedule more demanding broadcast sessions on dedicated workers

### Dynamic Scaling Triggers

The Instance Manager should implement a decision function that evaluates these metrics:

```javascript
function shouldScaleWorkers(currentMetrics) {
  // Scale UP conditions
  if (messageQueue.depth > QUEUE_THRESHOLD ||
      avgCpuUtilization > 70 ||
      avgMemoryUtilization > 80 ||
      sessionCount/workerCount > MAX_SESSIONS_PER_WORKER) {
    return 'scale_up';
  }
  
  // Scale DOWN conditions
  if (messageQueue.depth < LOW_QUEUE_THRESHOLD &&
      avgCpuUtilization < 30 &&
      avgMemoryUtilization < 40 &&
      (sessionCount/workerCount) < MIN_SESSIONS_PER_WORKER) {
    return 'scale_down';
  }
  
  return 'maintain';
}
```

### Implementation Approaches

1. **Kubernetes-based scaling (Preferred)**:
   * Create custom resource definitions for WhatsApp sessions
   * Deploy worker pods with resource limits matching session requirements
   * Use Horizontal Pod Autoscaler with custom metrics

2. **VM + Container Orchestration**:
   * Deploy worker VMs with Docker/containerd
   * Set resource limits per container
   * Auto-scale using VM instance groups/auto scaling sets

3. **Serverless with Constraints**:
   * AWS ECS/Fargate with appropriate memory configurations
   * Azure Container Instances with custom scaling logic

## Session Affinity and Task Routing

Session affinity is critical for performance and resource efficiency. Tasks related to the same WhatsApp session must be routed to the worker that has the authenticated session:

```javascript
// Pseudocode for routing tasks to workers
async function routeTask(task) {
  const sessionId = task.sessionId;
  let workerId = sessionToWorkerMap.get(sessionId);
  
  if (!workerId || !workerHealthMap.get(workerId)) {
    // Session not assigned or worker unhealthy
    workerId = await allocateSessionToWorker(sessionId);
    sessionToWorkerMap.set(sessionId, workerId);
  }
  
  // Send task to appropriate queue for that worker
  await messageQueue.send(`worker.${workerId}`, task);
}
```

The session allocation strategy should consider:

1. Current session count per worker
2. Resource usage of workers (CPU, memory utilization)
3. Session activity patterns 
4. Session relationships (group admins from same business → same worker)

## Implementation Path

### Phase 1: Task Queue and Scheduler Extraction

1. Modify the existing `scheduler.service.js` to use a message broker
2. Implement simple in-process worker pool
3. Define task structure for different operation types

```javascript
// Example task structure
{
  taskId: "broadcast_1234567890",
  sessionId: "8186852257",
  type: "BROADCAST",
  payload: {
    content: { /* message content */ },
    targetGroups: ["123456789@g.us", /* more groups */]
  },
  priority: "HIGH",
  timestamp: 1678912345678
}
```

### Phase 2: In-Process Worker Pools

1. Modify `threadPool.service.js` to create specialized worker pools
2. Implement session affinity within process
3. Add monitoring and metrics collection

### Phase 3: Distributed Workers

1. Extract worker logic to standalone processes
2. Implement Redis for session sharing
3. Deploy monitoring and autoscaling infrastructure

## Monitoring and Health Management

Implement comprehensive monitoring to tune your scaling strategy:

```javascript
// Collect worker metrics periodically
const workerMetrics = [
  'activeSessionCount',
  'memoryUsage',
  'cpuUtilization',
  'messageThroughput',
  'queueDepth',
  'responseLatency'
];
```

Use these metrics to:
1. Identify overloaded workers
2. Detect session performance issues
3. Optimize allocation strategy
4. Predict scaling needs

## Authentication Data Management

* Store WhatsApp session files in Redis with TTL
* Implement secure retrieval by workers with session-specific keys
* Implement versioning to detect outdated session data
* Perform periodic backup to persistent storage

## Security Considerations

* All communication between components is encrypted.
* Access to Redis is protected by authentication and authorization mechanisms.
* The authentication data is encrypted at rest and in transit.
* Regular security audits are conducted to identify and address vulnerabilities.

## Conclusion

The Master-Worker architecture with dynamic scaling provides a robust solution for WhatsApp integration at scale. This approach balances resource efficiency with operational reliability, allowing for controlled growth as user demand increases.
