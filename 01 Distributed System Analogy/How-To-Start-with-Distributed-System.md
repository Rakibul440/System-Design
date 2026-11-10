# System Design Concepts

## How To Start With Distributed System

---

## 1. Vertical Scaling

**Pizza analogy:** You have one chef. Business grows. You tell him to work faster, buy him better knives, and give him a bigger oven. Same chef — more power.

**Real engineering:** Vertical scaling means upgrading the hardware of a single server — more CPU cores, more RAM, faster SSD. Your application runs on one machine; you just make that machine stronger.

**Real-world example:** Early-stage startups often start on a single PostgreSQL instance. When queries slow down, the first move is to upgrade from a 4-core RDS instance to a 32-core one. Twitter did this in its early days — a single MySQL server was repeatedly upgraded before they were forced to rethink architecture entirely.

**When it breaks down:** Every machine has a hardware ceiling. A 128-core server costs exponentially more than 2× a 64-core. And it's still one machine — if it goes down, everything goes down.

**Interview note:** Always mention the ceiling and the single point of failure.

---

## 2. Preprocessing

**Pizza analogy:** You prepare all the dough and chop all the toppings at 4 AM — before the lunch rush hits. When orders come in at noon, the chef just assembles and bakes. Work done in advance = faster delivery.

**Real engineering:** Move expensive computation away from the request path. Run it as a background job during off-peak hours and store the result. When a user requests it, you just serve the precomputed result — no live computation needed.

**Real-world example:** YouTube doesn't transcode your video while you watch it. When a video is uploaded, background workers immediately start generating 360p, 480p, 720p, and 1080p versions. By the time you click play, all versions are ready. Same idea: Netflix pre-generates thumbnail images for every video at multiple sizes so the homepage loads in milliseconds.

**Another example:** E-commerce sites pre-generate "Best Sellers" and "Recommended for You" lists via nightly batch jobs — not on every page load.

**Interview note:** Mention cron jobs, message queues (Kafka, SQS), and the trade-off — data might be slightly stale.

---

## 3. Backup Servers (Redundancy)

**Pizza analogy:** You hire a backup chef who does nothing all day — but the moment your main chef calls in sick, he takes over. The kitchen never shuts down.

**Real engineering:** Every critical component should have a standby replica. If the primary fails, the system automatically fails over to the backup with zero (or minimal) downtime. This is called eliminating the **Single Point of Failure (SPOF)**.

**Real-world example:** AWS RDS Multi-AZ deployment. Your primary database runs in us-east-1a. A synchronous replica runs in us-east-1b. If the primary goes down — hardware failure, AZ outage — AWS automatically promotes the replica to primary within 60–120 seconds. Applications reconnect and continue. Users may see a brief slowdown; they never see a crash.

**Another example:** DNS itself is redundant by design — every domain has multiple NS records. If one nameserver goes down, queries route to another automatically.

**Interview note:** Distinguish between hot standby (running, ready instantly), warm standby (running but needs a few seconds to promote), and cold standby (off, needs manual start).

---

## 4. Horizontal Scaling

**Pizza analogy:** Instead of making one chef superhuman, you hire 10 chefs. Orders double? Hire 10 more. No single chef needs to be extraordinary.

**Real engineering:** Add more machines of the same type, all running the same application. A load balancer distributes incoming requests across all of them. When traffic spikes, spin up more instances. When it drops, spin them down.

**Real-world example:** Netflix runs hundreds of thousands of EC2 instances. During peak hours (8–10 PM), they auto-scale up. At 3 AM, they scale down to save cost. Each instance runs the same streaming microservice code. No single instance is special — if 50 of them crash, the other 450 absorb the load.

**Another example:** When WhatsApp was acquired by Facebook (2014), it was handling 450 million users with only 32 engineers — largely because their Erlang servers scaled horizontally with minimal operational overhead.

**Interview note:** Horizontal scaling requires the app to be stateless — the server cannot store session data locally, because the next request might go to a different server. Session state goes in Redis or a database.

---

## 5. Microservices

**Pizza analogy:** Instead of one giant kitchen doing everything, you split into specialized teams: the Pizza Team, the Garlic Bread Team, the Drinks Team. Each team can be expanded, retrained, or replaced without touching the others.

**Real engineering:** Break a monolithic application into small, independently deployable services. Each service owns one domain — User Service, Order Service, Payment Service, Notification Service. They communicate via APIs or message queues. Each can be written in a different language, scaled independently, and deployed without a full system restart.

**Real-world example:** Amazon is the textbook case. Around 2002, Jeff Bezos issued his famous "API Mandate" — every team must expose its data through APIs, with no direct database sharing. This forced Amazon to decompose into services. Today, a single Amazon page load triggers calls to hundreds of microservices: recommendations, inventory, pricing, reviews, ads — all independent.

**Another example:** Uber split from a monolith into microservices when their monolith's deployment took 4+ hours and a bug in the billing code could crash trip tracking. After the split, each service deploys independently in minutes.

**Interview note:** Mention the trade-off — microservices add operational complexity (service discovery, distributed tracing, network latency). They're not always the right answer for small teams.

---

## 6. Distributed Systems

**Pizza analogy:** You open a second shop in a new neighborhood. Local customers get faster delivery. If one shop catches fire, the other keeps running.

**Real engineering:** Distribute your system across multiple machines, data centers, or geographic regions. This achieves two things: (1) **fault tolerance** — no single location failure kills the whole system, and (2) **lower latency** — users are served from a node physically close to them.

**Real-world example:** Cloudflare operates 300+ data centers worldwide. When you visit a website behind Cloudflare, your request is served from the nearest edge node — maybe Mumbai if you're in Kolkata — not from a server in Virginia. This cuts latency from 200ms to 10ms. If the Mumbai data center goes offline, traffic automatically reroutes to Singapore or Chennai.

**Another example:** Google Spanner is a globally distributed SQL database. It stores data across data centers in multiple continents, with globally consistent transactions. Banks use this to ensure a transaction is never lost even if an entire region goes down.

**Interview note:** Bring up the CAP theorem — a distributed system can guarantee at most two of: Consistency, Availability, Partition tolerance. This is a critical trade-off in every distributed design.

---

## 7. Load Balancing

**Pizza analogy:** A manager stands at the entrance. When a customer arrives, he checks both shops — "Shop A has 3 orders pending, Shop B has 1" — and sends the customer to Shop B. No shop gets overwhelmed while another sits idle.

**Real engineering:** A load balancer sits in front of your server fleet. It receives every incoming request and decides which server handles it, using algorithms like:

- **Round-robin:** rotate through servers in order
- **Least connections:** send to the server with fewest active requests
- **IP hash:** same user always goes to same server (useful for sessions)

**Real-world example:** Every major tech company uses this. When you type google.com, your request hits Google's load balancer (built on Maglev, their custom LB software). It routes you to one of thousands of frontend servers based on health checks and load. If a server is unhealthy, Maglev stops sending traffic to it — you never notice. AWS ALB (Application Load Balancer) and Nginx do this for most web applications.

**Another example:** During IPL live streams on Hotstar, traffic spikes to 20–25 million concurrent users in seconds. Their load balancers auto-detect overloaded servers and reroute within milliseconds.

**Interview note:** Mention Layer 4 LB (routes by IP/TCP, fast) vs Layer 7 LB (routes by HTTP content, smarter — can send /api to one cluster and /static to another).

---

## 8. Decoupling

**Pizza analogy:** The delivery agent picks up any food — pizza, burger, biryani — without knowing how it was made. The shop doesn't need to reprogram the delivery app when it adds a new item. They talk through one contract: "here is a box, deliver it to this address."

**Real engineering:** Components should interact through well-defined interfaces (APIs, message queues, events) — not by directly calling each other's internals. If Service A directly calls Service B's internal function, changing B breaks A. If they talk through a message queue (Kafka, SQS), A just publishes an event and doesn't care what happens next.

**Real-world example:** When you place an order on Swiggy/Zomato, the Order Service publishes an event: "Order #XYZ placed." Multiple downstream services listen independently:

- Payment Service charges your card
- Notification Service sends you a confirmation SMS
- Restaurant Service alerts the kitchen
- Analytics Service logs the order

None of these services know about each other. The Order Service doesn't call Payment directly. If the Notification Service goes down, orders still process — the notification just comes 2 minutes late when the service recovers.

**Interview note:** Kafka and RabbitMQ are the most common tools here. This pattern is called **Event-Driven Architecture**.

---

## 9. Logging & Metrics

**Pizza analogy:** You track how long each order takes at each shop. One day you notice Shop B's average time jumped from 12 minutes to 45 minutes. You investigate — broken oven. Without tracking, you'd only know when customers started complaining.

**Real engineering:** Every service should emit:

- **Logs** — a text record of every significant event ("User 42 logged in", "Payment failed: insufficient funds")
- **Metrics** — numeric time-series data (request count, error rate, p99 latency, CPU %)
- **Traces** — the end-to-end journey of a single request across all services

These three together are called **Observability**.

**Real-world example:** At Netflix, every microservice reports metrics to their Atlas monitoring system every second. When error rate on the Playback Service crosses 0.1%, an automated alert fires before any customer can even tweet a complaint. Engineers get a PagerDuty alert, open a Grafana dashboard, and see exactly which server in which region started dropping requests and when.

**Another example:** Google's SRE teams use **SLOs (Service Level Objectives)** — "99.9% of requests must complete in under 200ms." Metrics pipelines track this in real-time. If the error budget is burning too fast, the team pauses feature releases and focuses on reliability.

**Tools:** Prometheus + Grafana (metrics), ELK Stack / Loki (logs), Jaeger / Zipkin / Datadog APM (traces).

**Interview note:** Mention the difference between logs, metrics, and traces — interviewers love when you use the word "observability" and explain all three pillars.

---

## 10. Extensibility

**Pizza analogy:** You build your kitchen, delivery app, and billing system so generically that adding burgers to the menu only means adding one new item to a catalog. You don't rebuild the kitchen, retrain every driver, or rewrite the app.

**Real engineering:** Design systems so that new features are additions, not modifications. This maps to the **Open/Closed Principle** — open for extension, closed for modification. Use plugin architectures, feature flags, and clean interfaces so new functionality slots in without touching existing code.

**Real-world example:** Stripe's payment API is a masterclass in extensibility. When they added SEPA Direct Debit, Apple Pay, and Buy Now Pay Later, their core charge API didn't change. Merchants who built on Stripe automatically got access to new payment methods — their integration required zero changes. Stripe added new "payment method plugins" behind the same interface.

**Another example:** Shopify's App Store. The core Shopify platform never changes — but merchants can add loyalty programs, review widgets, and shipping integrations via plugins. Each app connects through the same extension API.

**Another example:** VS Code is entirely plugin-based. The core editor is minimal. Every language feature (Python IntelliSense, Go debugger) is an extension. Microsoft can add Rust support without touching the core codebase.

**Interview note:** Connect this to SOLID principles (especially Open/Closed). Mention feature flags (LaunchDarkly) as a runtime extensibility pattern — ship code but control who sees it.

---

## Quick Reference Table

| # | Concept | Core idea | Real-world tool/example |
| --- | --- | --- | --- |
| 1 | Vertical scaling | Bigger machine | AWS RDS instance upgrade |
| 2 | Preprocessing | Work in advance | YouTube transcoding, Netflix thumbnails |
| 3 | Backup servers | No SPOF | AWS RDS Multi-AZ |
| 4 | Horizontal scaling | More machines | Netflix EC2 auto-scaling |
| 5 | Microservices | Split by domain | Amazon API Mandate, Uber |
| 6 | Distributed systems | Spread across regions | Cloudflare CDN, Google Spanner |
| 7 | Load balancing | Smart traffic routing | AWS ALB, Nginx, Google Maglev |
| 8 | Decoupling | Talk through contracts | Kafka, Swiggy order events |
| 9 | Logging & metrics | Observe everything | Prometheus + Grafana, Netflix Atlas |
| 10 | Extensibility | Build for future change | Stripe API, Shopify App Store, VS Code |
