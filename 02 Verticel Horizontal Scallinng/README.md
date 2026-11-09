# Vertical vs Horizontal Scaling

> A practical system-design guide to scaling applications for higher
> traffic, throughput, availability, and growth.

## Table of contents

- [1. What is scalability?](#1-what-is-scalability)
- [2. Vertical scaling (scale up)](#2-vertical-scaling-scale-up)
- [3. Horizontal scaling (scale out)](#3-horizontal-scaling-scale-out)
- [4. Side-by-side comparison](#4-side-by-side-comparison)
- [5. How they work together](#5-how-they-work-together)
- [6. Practical Node.js example](#6-practical-nodejs-example)
- [7. Important concepts and
  pitfalls](#7-important-concepts-and-pitfalls)
- [8. When should you choose each?](#8-when-should-you-choose-each)
- [9. Interview revision](#9-interview-revision)
- [10. Further learning](#10-further-learning)

------------------------------------------------------------------------

## 1. What is scalability?

**Scalability** is a system's ability to handle an increase in workload
by adding resources or improving how existing resources are used, while
maintaining acceptable performance.

Workload can grow in several ways:

- More users online at the same time.
- More requests per second.
- Larger databases and files.
- More background jobs.
- More concurrent connections.
- Higher traffic during predictable peak periods.

A system is not scalable merely because it has many servers. It must be
able to use additional resources effectively without creating a new
bottleneck elsewhere.

### Useful performance terms

| Term         | Meaning                                                            |
|--------------|--------------------------------------------------------------------|
| Throughput   | Amount of work completed per unit of time, such as requests/second |
| Latency      | Time taken to complete one operation                               |
| Concurrency  | Number of operations in progress at the same time                  |
| Availability | Degree to which a service is operational and accessible            |
| Bottleneck   | The component limiting overall system performance                  |
| Capacity     | Maximum workload a system can handle under stated conditions       |

### Pizza-shop analogy

A small pizza shop initially has one chef and one oven. As orders grow,
the owner has two broad choices:

1.  Upgrade the existing kitchen so the same chef and oven can do more
    work.
2.  Add more kitchens, chefs, and ovens so work can happen in parallel.

These correspond to **vertical scaling** and **horizontal scaling**.

------------------------------------------------------------------------

## 2. Vertical scaling (scale up)

### 2.1 Definition

**Vertical scaling** means increasing the resources available to one
machine or instance.

Typical upgrades include:

- More CPU cores or faster CPUs.
- More RAM.
- Faster or larger storage.
- Higher network capacity.
- A larger cloud VM or database instance.

The number of application instances does not increase as part of the
scaling operation. The existing machine becomes more powerful.

### 2.2 Animated diagram

![Animated vertical scaling: one server gets more
resources](./vertical-scaling.svg)

### 2.3 Before and after

``` text
BEFORE                                  AFTER

       Requests                              Requests
           |                                     |
           v                                     v
   +----------------+                    +------------------+
   |   Server A     |                    |    Server A      |
   |                |     UPGRADE        |                  |
   |  2 vCPU        |  ------------->    |  16 vCPU         |
   |  4 GB RAM      |                    |  64 GB RAM       |
   |  Basic disk    |                    |  Faster storage  |
   +----------------+                    +------------------+
           |                                     |
           v                                     v
        Database                              Database

   One application instance              Still one instance
```

The resource values are illustrative, not a universal upgrade path.

### 2.4 How it works

1.  Measure the workload and identify the bottleneck.
2.  Determine whether CPU, memory, disk I/O, or network capacity is
    limiting performance.
3.  Select a larger machine or instance type.
4.  Move or resize the workload.
5.  Test latency, throughput, and resource utilization again.

A CPU upgrade will not solve a database query that is slow because it
lacks an index. Scaling should follow measurement, not guesswork.

### 2.5 Real-world example

Suppose a Node.js API runs on a cloud VM:

``` text
Initial instance:
- 2 vCPU
- 4 GB RAM
- 50 GB disk

Larger instance:
- 8 vCPU
- 16 GB RAM
- 100 GB disk
```

The application remains on one instance, but it has more compute and
memory capacity.

### 2.6 Advantages

- Usually straightforward to understand and operate.
- Often requires fewer application architecture changes.
- Can improve performance quickly when the bottleneck is a resource
  limit.
- Avoids distributing application state across many instances.
- Can be a good first scaling step for a small or medium workload.

### 2.7 Limitations

- A machine has a maximum size.
- Larger instances can become disproportionately expensive.
- Resizing may require a restart or maintenance window, depending on the
  platform.
- One machine can remain a single point of failure.
- A bigger application server does not automatically fix a database,
  network, or external-service bottleneck.

### 2.8 When is vertical scaling useful?

Choose it when:

- The application is still relatively small.
- A single instance is easy to operate.
- The workload is limited by a resource that can be upgraded.
- You need a quick capacity increase without redesigning the
  application.
- Your database or workload has practical constraints that make
  distributing it difficult.

------------------------------------------------------------------------

## 3. Horizontal scaling (scale out)

### 3.1 Definition

**Horizontal scaling** means increasing capacity by adding more machines
or application instances and distributing work among them.

Instead of making one server bigger, run multiple servers that can
handle requests in parallel.

### 3.2 Animated diagram

![Animated horizontal scaling: requests are distributed across multiple
servers](./horizontal-scaling.svg)

### 3.3 Before and after

``` text
BEFORE

             Users
               |
               v
        +--------------+
        |  Server A    |
        |  Application |
        +--------------+
               |
               v
            Database


AFTER

             Users
               |
               v
        +--------------+
        | Load Balancer|
        +--------------+
          /     |     \
         v      v      v
      +------+ +------+ +------+
      | A    | | B    | | C    |
      | App  | | App  | | App  |
      +------+ +------+ +------+
          \      |      /
           \     |     /
                v
          Shared data layer
```

### 3.4 How it works

1.  Deploy the same application to multiple instances.
2.  Put a load balancer or routing layer in front of them.
3.  Distribute incoming requests across healthy instances.
4.  Monitor each instance and remove unhealthy instances from routing.
5.  Add or remove instances as demand changes.

A cloud platform may do this manually, through an instance group, or
automatically using autoscaling rules.

### 3.5 Load-balancing strategies

| Strategy             | How it routes traffic                              | Typical consideration                                |
|----------------------|----------------------------------------------------|------------------------------------------------------|
| Round robin          | Sends requests to instances in sequence            | Simple, but requests may have different costs        |
| Weighted round robin | Sends more traffic to higher-weight instances      | Useful when instance capacities differ               |
| Least connections    | Prefers the instance with fewer active connections | Helpful when connection duration varies              |
| IP hash              | Uses client IP to choose an instance               | Can provide affinity, but distribution may be uneven |
| Least response time  | Prefers instances responding more quickly          | Requires reliable response-time measurements         |

The best strategy depends on workload, connection behaviour, and
infrastructure.

### 3.6 Stateless application design

Horizontal scaling is easiest when any healthy application instance can
handle any request.

Consider this sequence:

``` text
Request 1 -> Server A
Request 2 -> Server B
```

If the user's session exists only in Server A's local memory, Server B
may not know who the user is.

A common solution is to keep shared state outside individual application
instances:

``` text
             +-------------+
             | Load Balancer|
             +-------------+
                /       \
               v         v
          +--------+ +--------+
          |Server A| |Server B|
          +--------+ +--------+
               \       /
                \     /
                 v   v
              +--------+
              | Redis  |
              | Shared |
              | State  |
              +--------+
```

Possible approaches:

- Store sessions in Redis or a shared session store.
- Use signed access tokens that each instance can validate.
- Store durable application data in a shared database or service.
- Store uploaded files in shared object storage rather than local
  instance disks.

**Stateless does not mean the whole application has no state.** It means
an application instance does not rely on private local state that
another instance cannot access.

### 3.7 Advantages

- Capacity can grow by adding instances.
- Work can be processed in parallel.
- Individual application instances can be replaced or removed.
- Supports elastic capacity and autoscaling.
- Can improve availability when instances are distributed across failure
  domains.

### 3.8 Limitations

- Requires traffic distribution and health checks.
- Shared databases, caches, queues, and storage can become bottlenecks.
- Session and other shared-state handling require careful design.
- Network calls introduce latency and failure modes.
- Distributed debugging and deployment become more complex.
- More instances do not guarantee linear performance improvement.

### 3.9 When is horizontal scaling useful?

Choose it when:

- Traffic or concurrency is growing beyond one machine's practical
  capacity.
- Requests can be processed independently across instances.
- You need elastic capacity.
- You want to spread application instances across availability zones or
  machines.
- You need to replace or maintain instances without taking the whole
  service offline.

------------------------------------------------------------------------

## 4. Side-by-side comparison

| Dimension            | Vertical scaling                                    | Horizontal scaling                                           |
|----------------------|-----------------------------------------------------|--------------------------------------------------------------|
| Basic idea           | Make one machine bigger                             | Add more machines                                            |
| Common name          | Scale up                                            | Scale out                                                    |
| Number of instances  | Usually unchanged                                   | Increases                                                    |
| Capacity limit       | Limited by the largest machine                      | Can grow by adding instances, subject to architecture limits |
| Application changes  | Often fewer                                         | Often more                                                   |
| State management     | Usually simpler                                     | Shared state must be designed carefully                      |
| Traffic distribution | Not necessarily needed                              | Usually needs load balancing or routing                      |
| Fault tolerance      | Does not inherently remove a single-machine failure | Can improve resilience if failures are isolated              |
| Cost model           | Pay for a larger instance                           | Pay for multiple instances and supporting infrastructure     |
| Operations           | Often simpler                                       | More deployment and monitoring complexity                    |
| Typical use          | Early growth, resource-bound workloads              | High traffic, concurrency, elastic workloads                 |

### The key distinction

``` text
VERTICAL:                         HORIZONTAL:

One bigger machine                More machines

   +----------+                     +------+ +------+ +------+
   |          |                     | A    | | B    | | C    |
   |  BIGGER  |                     +------+ +------+ +------+
   |  SERVER  |                         \      |      /
   |          |                          \     |     /
   +----------+                           Load Balancer
```

------------------------------------------------------------------------

## 5. How they work together

Vertical and horizontal scaling are **not mutually exclusive**. Real
systems often use both.

For example:

1.  Start with one small application instance.
2.  Increase its CPU and memory when the workload grows.
3.  When one machine is no longer sufficient or resilient enough, run
    multiple instances.
4.  Choose instance sizes that balance cost and capacity.
5.  Scale the database, cache, and background workers according to their
    own bottlenecks.

### Combined architecture

``` text
                         Users
                           |
                           v
                    +-------------+
                    | Load Balancer|
                    +-------------+
                      /         \
                     v           v
              +------------+ +------------+
              | App Server | | App Server |
              | A          | | B          |
              | 8 vCPU     | | 8 vCPU     |
              | 16 GB RAM  | | 16 GB RAM  |
              +------------+ +------------+
                     \           /
                      \         /
                       v       v
                  +-------------+
                  | Shared Data |
                  | / Services  |
                  +-------------+
```

This design uses:

- **Horizontal scaling:** two application instances.
- **Vertical sizing:** each instance has a chosen CPU and RAM capacity.
- **Load balancing:** requests are distributed between instances.
- **Shared state:** both instances access common data services.

### A practical growth path

| Stage | Architecture                          | Main reason                           |
|-------|---------------------------------------|---------------------------------------|
| 1     | One small server                      | Start simply                          |
| 2     | One larger server                     | Handle a resource bottleneck          |
| 3     | Multiple application instances        | Increase parallel capacity            |
| 4     | Load balancer + health checks         | Route traffic to healthy instances    |
| 5     | Shared cache and object storage       | Avoid instance-local state            |
| 6     | Database optimization and scaling     | Address data-layer bottlenecks        |
| 7     | Autoscaling and multi-zone deployment | Adapt capacity and improve resilience |

Do not jump to the final stage without evidence that the application
needs it.

------------------------------------------------------------------------

## 6. Practical Node.js example

Imagine an Express API deployed as one process:

``` js
import express from "express";

const app = express();

app.get("/health", (req, res) => {
  res.json({ status: "ok" });
});

app.get("/api/posts", async (req, res) => {
  // Fetch posts from the database.
  res.json([]);
});

app.listen(3000, () => {
  console.log("API listening on port 3000");
});
```

### Vertical scaling

Deploy this same application on a larger VM. The code may not need to
change, but the machine has more CPU and RAM.

### Horizontal scaling

Run multiple copies of the application:

``` text
Server A: node server.js
Server B: node server.js
Server C: node server.js
```

Place a load balancer in front of them:

``` text
                  Client
                    |
                    v
              Nginx / LB
              /    |    \
             v     v     v
          :3001  :3002  :3003
          Node   Node   Node
```

In production, each instance should have:

- A health endpoint.
- Centralized logs.
- Shared session/state storage where needed.
- Externalized configuration and secrets.
- Graceful shutdown handling.
- Database connection limits appropriate to the total number of
  instances.

**Database connection warning:** If every instance opens too many
database connections, adding application servers can overload the
database. Use connection pooling and size the pool across all instances.

------------------------------------------------------------------------

## 7. Important concepts and pitfalls

### 7.1 A load balancer is not the same as horizontal scaling

A load balancer distributes traffic. It does not create additional
application capacity by itself. It becomes useful when there are
multiple destinations to route to.

### 7.2 Horizontal scaling does not automatically make a system highly available

If all instances run on one physical host, one host failure can take
them all down. Spread instances across independent failure domains when
availability requires it.

### 7.3 The database may become the bottleneck

Adding API instances can increase database traffic. The database may
need query optimization, indexing, caching, read replicas, partitioning,
or sharding depending on the workload.

### 7.4 Not every workload scales horizontally

Some workloads depend on shared mutable state, ordered processing, or
exclusive access to a resource. They may need coordination,
partitioning, or a different architecture before adding instances helps.

### 7.5 Measure before scaling

Track:

- Request rate and throughput.
- p50, p95, and p99 latency.
- Error rate.
- CPU and memory utilization.
- Database query latency and connection usage.
- Queue depth and job processing time.

Averages alone can hide slow requests. Tail latency (p95/p99) is often
important for user experience.

### 7.6 Autoscaling

Autoscaling automatically adds or removes instances based on configured
signals, such as CPU utilization, request rate, or queue depth.

``` text
Demand increases
      |
      v
Metrics cross threshold
      |
      v
Autoscaler adds instances
      |
      v
More capacity available

Demand decreases
      |
      v
Autoscaler removes excess instances
```

Autoscaling needs sensible minimums, maximums, cooldowns, health checks,
and capacity planning. It cannot react instantly to every traffic spike.

------------------------------------------------------------------------

## 8. When should you choose each?

| Situation                                           | Reasonable starting choice                           |
|-----------------------------------------------------|------------------------------------------------------|
| Small application with low traffic                  | One appropriately sized instance                     |
| CPU or RAM is saturated on one instance             | Vertical scaling, after confirming the bottleneck    |
| Requests are independent and traffic is growing     | Horizontal scaling                                   |
| Need maintenance without taking the service offline | Multiple instances across failure domains            |
| Database is slow due to missing indexes             | Optimize queries/indexes before adding app servers   |
| Traffic has predictable peaks                       | Autoscaling, caching, and/or precomputation          |
| One large in-memory workload                        | Vertical scaling may be simpler, depending on limits |
| Global users need low latency                       | Regional deployment and routing may be appropriate   |

**Rule of thumb:** Start with the simplest architecture that meets the
service's performance and availability targets. Scale the component that
measurements show is limiting the system.

------------------------------------------------------------------------

## 9. Interview revision

### Q1. What is vertical scaling?

Increasing the resources of one machine, such as CPU, RAM, or storage.
It is also called scale up.

### Q2. What is horizontal scaling?

Adding more machines or instances and distributing workload across them.
It is also called scale out.

### Q3. Which one is better?

Neither is universally better. The choice depends on workload, cost,
architecture, availability requirements, and operational complexity.
Many production systems combine both.

### Q4. Why is statelessness useful for horizontal scaling?

Any healthy instance can handle a request without depending on session
data stored only in another instance's local memory.

### Q5. Does horizontal scaling guarantee high availability?

No. Instances must be placed across independent failure domains,
health-checked, and supported by resilient dependencies.

### Q6. What is the role of a load balancer?

It routes incoming traffic across available instances according to a
routing policy and can stop sending traffic to unhealthy instances.

### Q7. What happens after adding more application servers?

The database, cache, network, or a downstream service may become the
next bottleneck. Measure the whole request path.

### Q8. Can vertical and horizontal scaling be combined?

Yes. Run multiple instances and choose a suitable size for each
instance.

------------------------------------------------------------------------

## 10. Further learning

After understanding these two scaling approaches, study:

1.  Client-server architecture
2.  Reverse proxy and load balancers
3.  Stateless services and session management
4.  Caching (Redis)
5.  Database indexing and query optimization
6.  Replication and read replicas
7.  Database sharding and partitioning
8.  Message queues and asynchronous processing
9.  Autoscaling and health checks
10. Availability zones and fault tolerance
11. CAP theorem and consistency models
12. Monitoring, logging, and distributed tracing

### Suggested hands-on exercise

Use an existing Node.js application:

1.  Deploy one instance and measure its baseline.
2.  Increase the instance size and compare the measurements.
3.  Run two or three instances.
4.  Put Nginx or a cloud load balancer in front.
5.  Move sessions and uploaded files out of local instance storage.
6.  Run a load test and compare throughput, p95 latency, and error rate.
7.  Observe whether the database becomes the bottleneck.

The goal is not merely to make the diagram look distributed. The goal is
to measure what changed and understand why.

------------------------------------------------------------------------

## Final takeaway

- **Vertical scaling:** Make one machine stronger.
- **Horizontal scaling:** Add more machines.
- **Combined scaling:** Use appropriately sized machines and multiple
  instances to balance performance, cost, and resilience.

> Good system design is not about using the most servers. It is about
> identifying the bottleneck and choosing the simplest architecture that
> meets the required performance, reliability, and cost targets.
