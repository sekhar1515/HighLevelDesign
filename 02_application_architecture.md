# Application Architecture & System Observability: How Modern Systems are Built

To build scalable, resilient, and maintainable software systems, we must transition from running a single application on a single server to deploying a distributed architecture. 

The diagram below outlines the lifecycle of a modern web application: how code is deployed, how traffic is routed and scaled, and how we monitor system health in production.

---

## 1. The Big Picture: Analyzing the Architecture Diagram

Let's trace the flows in the architecture diagram:

```
                  +-------------------------+
                  |  BUILD & DEPLOY SYSTEM  |
                  +-------------------------+
                               || (Deploys code)
                               \/
+------+   HTTP   +---------------+        +------------+        +-------------+
| User |=========>| Load Balancer |=======>| Server #1  |=======>|   Storage   |
+------+          +---------------+        +------------+        | (DB / Disk) |
                                           | Server #2  |        +-------------+
                                           +------------+
                                                 || 
                                                 || Writes logs & metrics
                                                 \/
                                           +------------+
                                           |  Logging   |
                                           +------------+
                                                 || Aggregates to
                                                 \/
                                           +------------+
                                           |  Metrics   |
                                           +------------+
                                                 || Triggers
                                                 \/
                                           +------------+
                                           |  Alerts    | [RED PENTAGON]
                                           +------------+
```

There are three main flows happening in this architecture:
1. **The Development & Deployment Flow**: Code is built and shipped to the servers.
2. **The User Traffic & Execution Flow**: Users send requests, which are balanced across servers, and servers interact with storage.
3. **The Observability Flow**: The servers emit logs and metrics, which are aggregated and monitored to trigger alerts.

---

## 2. Build & Deploy: How Code Reaches the Server

Before a user can interact with an application, the developer's source code must be compiled, tested, packaged, and running on a server.

```
+-----------+     Push     +--------------+     Package     +------------------+
| Developer |============> | Build Engine |==============>  | Deploy Engine    |
| (Code)    |              | (CI / Tests) |                 | (Canary/Rolling) |
+-----------+              +--------------+                 +------------------+
                                                                     ||
                                                                     \/
                                                            +------------------+
                                                            | Running Servers  |
                                                            +------------------+
```

* **Continuous Integration (CI / Build)**: When developers push code, an automated build server runs tests and compiles the code into a deployable artifact (e.g., a Docker image, executable, or zip archive).
* **Continuous Deployment (CD / Deploy)**: The deploy system updates the servers with the new artifact using deployment strategies to prevent downtime:
  * **Rolling Deployment**: Gradually replacing old instances with new ones one by one.
  * **Blue-Green Deployment**: Deploying the new version (Green) alongside the old version (Blue), then flipping a router switch to direct all traffic to Green.
  * **Canary Deployment**: Directing a tiny percentage (e.g., 5%) of real traffic to the new version to test it in production before a full rollout.

---

## 3. Scaling & Routing: Handling Traffic at Scale

Once the application is running, users begin sending requests. As traffic grows, we must route requests efficiently and scale our compute resources.

### The Entry Point: The Load Balancer (LB)
A user's browser does not talk directly to an application server. Instead, it hits a **Load Balancer**, which acts as a traffic director.
1. **Traffic Distribution**: It intercepts requests and routes them to healthy servers using algorithms like **Round Robin** (sequential), **Least Connections** (routing to the least busy server), or **IP Hash** (sticky routing based on client IP).
2. **Health Checks**: The LB constantly checks the status of servers. If a server crashes, the LB detects the failure and stops routing traffic to it.
3. **SSL Termination**: Decrypting HTTPS traffic requires CPU overhead. The load balancer can decrypt the traffic at the edge and pass unencrypted HTTP traffic internally, saving app server CPU.

### Scaling Strategies: Vertical vs. Horizontal
```
       Vertical Scaling (Scale Up)               Horizontal Scaling (Scale Out)
          +-----------------+                 +---------+  +---------+  +---------+
          |                 |                 |         |  |         |  |         |
          |  Server (Huge)  |                 | Server  |  | Server  |  | Server  |
          |                 |                 |  (App)  |  |  (App)  |  |  (App)  |
          +-----------------+                 +---------+  +---------+  +---------+
```

* **Vertical Scaling (Scale Up)**: Adding more power (bigger CPU, more RAM, faster SSD) to a single machine.
  * *Pros*: Simple to set up; no changes to application architecture are needed.
  * *Cons*: Hard hardware limits; exponential cost curve; Single Point of Failure (SPOF).
* **Horizontal Scaling (Scale Out)**: Adding more machines (nodes) of standard, commodity size to your system.
  * *Pros*: Infinite scale potential; high availability (if one node crashes, others handle traffic); cost-efficient.
  * *Cons*: Requires stateless application design; data consistency across nodes is harder to maintain.

---

## 4. Storage: Stateless Compute vs. Stateful Persistence

To scale horizontally, application servers must be **stateless**. This means they do not store user sessions, uploaded files, or permanent records on their local disks. If a server is destroyed, no data should be lost. 

Instead, all state is delegated to the **Storage** layer.

```
       [ Stateless Compute Layer ]             [ Stateful Storage Layer ]
        +-------------------------+            +-------------------------+
        | App Server   App Server |===========>| Database / Disk Store   |
        | (Stateless)  (Stateless)|            | (Stateful persistence)  |
        +-------------------------+            +-------------------------+
```

* **Stateless Compute**: Servers can be spun up or down instantly in response to traffic spikes because they are identical clones.
* **Stateful Storage**: Data durability is managed by dedicated databases and storage engines:
  * **Relational Databases (RDBMS)**: (e.g., PostgreSQL, MySQL) for structured, transactional data requiring strong consistency.
  * **NoSQL Databases**: (e.g., MongoDB, DynamoDB) for unstructured or semi-structured data requiring high-throughput horizontal scale.
  * **Object Storage**: (e.g., AWS S3) for flat files, images, and videos.

---

## 5. Logging: Recording "What Happened" (High Detail)

Once our system is running and handling traffic, we must observe its health. The first layer of observability is **Logging**.

Logs represent a text record of discrete events that occurred within the system.

### Why not just use `print` / `console.log`?
Standard outputs are synchronous (blocking the execution thread), cannot easily be grouped by severity level, and lack formatting metadata necessary for automated search tools.

### Core Logging Concepts
1. **Log Levels**: Standardized categories to filter logs by importance:
   * `DEBUG`: Detailed diagnostic information for development.
   * `INFO`: Normal operational events (e.g., "User logged in").
   * `WARN`: Unexpected event occurred but the system recovered (e.g., "Database query took 2.5 seconds").
   * `ERROR`: A request failed due to a problem (e.g., "Payment gateway timed out").
   * `FATAL`: A critical error occurred causing a process/thread to crash.
2. **Structured Logging (JSON)**: Modern applications write logs as JSON objects rather than plain text. This allows downstream computers to parse, index, and query log data efficiently.
   * *Structured JSON*: `{"timestamp": "2026-07-15T07:50:00Z", "level": "ERROR", "message": "Payment failed", "userId": 123}`

### Popular Logging Stacks
* **Application Libraries**: Winston/Pino (Node.js), Logback/Log4j2 (Java), Loguru (Python).
* **The Log Pipeline**:
  * **ELK Stack**: **E**lasticsearch (indexes and stores logs), **L**ogstash (collects and parses logs), **K**ibana (visual interface to query logs).
  * **PLG Stack**: **P**romtail (scrapes local log files), **L**oki (stores and indexes logs cheaply), **G**rafana (visualizes search queries).

---

## 6. Metrics: Measuring Performance (Aggregated Numbers)

Logs provide high-fidelity details about single events, but they are expensive to store. To understand overall system health at a glance, we use **Metrics**. 

Metrics are numeric values aggregated over time (e.g., CPU utilization, memory usage, request counts).

```
Logs = "User 102 failed to checkout due to null pointer at 07:50:11" (Highly detailed, expensive)
Metrics = "Checkout Failure Rate = 1.2% at 07:50:00" (Numerical aggregation, lightweight)
```

### The 4 Golden Signals of Monitoring
1. **Latency**: The time it takes to service a request (e.g., 95th percentile response time is 180ms).
2. **Traffic**: The volume of demand on the system (e.g., HTTP requests per second).
3. **Errors**: The rate of requests that fail (e.g., percentage of HTTP 5xx responses).
4. **Saturation**: How close system resources are to full capacity (e.g., 85% CPU load, database pool exhaustion).

### Popular Metrics Tools
* **Prometheus**: An open-source metrics database that pulls numeric values from applications at set intervals.
* **Grafana**: The industry-standard tool for creating rich visual dashboards to graph metrics over time.

---

## 7. Alerting: Translating Data to Action

Metrics and logs are passive unless they trigger action. **Alerting** is the final step: automatically notifying engineers when metrics indicate an anomaly or failure.

### The Observability Pipeline in Action
```
[ Server processes traffic ]
           ||
           \/
[ Emits Metrics / Logs ] 
           ||
           \/
[ Prometheus scrapes / aggregates metrics ]
           || (Evaluates alerting rule)
           \/
[ Alerting Rule Triggered ] -> Sum of 5xx errors > 5% over 5 mins
           ||
           \/
[ Notification Engine (PagerDuty) ]
           || (Determines severity)
           \/
[ On-Call Engineer Paged ] (SMS / Call)
```

### Alerting Best Practices
* **Symptom-based Alerting**: Alert on issues that directly impact users (e.g., "checkout failure rate is high") rather than internal causes (e.g., "CPU utilization is 90%"). High CPU is fine if the user experience remains fast and error-free.
* **Preventing Alert Fatigue**: If alerting thresholds are set too low, engineers receive constant false-alarm pages. Over time, they will ignore alerts, leading to missed production failures.
* **Paging vs. Ticketing**:
  * **Severity - Page (P1)**: High-priority. Triggers tools like **PagerDuty** or **Opsgenie** to wake up the on-call engineer via phone call or SMS.
  * **Severity - Ticket (P2)**: Low-priority. Posts to a Slack channel or creates a Jira ticket to be investigated during business hours.
