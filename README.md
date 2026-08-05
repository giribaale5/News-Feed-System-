1. System Requirements :
   Functional Requirements :
     -> Post Creation: Users can publish posts containing text and media (images/videos).
     -> News Feed Generation: Users can view a personalized feed of posts generated from accounts they follow.
     -> User Relationship / Social Graph: Users can follow and unfollow other users.
     -> Feed Interactions: Users can like and comment on posts in their feed.

   Non-Functional Requirements :
     -> High Availability: Read-heavy system with high availability prioritized over strong consistency (eventual consistency for feed updates is acceptable).
     -> Low Latency: Fast feed loading ($\le 200\text{ ms}$ response time for feed generation).
     -> Scalability: System must scale to handle hundreds of millions of daily active users (DAUs) and fan-out spikes (e.g., celebrity posts).
     -> Reliability & Fault Tolerance: No single point of failure; gracefully degraded experience during system degradation.

2. High-Level Architecture & Component Flow :
        
                         [ Client ]
                              │
                              ▼
                          [ API Gateway / Load Balancer ]
                              │
                              ├───► [ Post Service ] ────► [ Relational DB (Posts) ]
                              │          │
                              │          └───────────────► [ Message Broker (Kafka) ] ───► [ Fanout Worker Service ]
                              │                                                                   │
                              ├───► [ User / Follow Service ] ──► [ Graph DB / Cache ]             │
                              │                                                                   ▼
                              └───► [ Feed Service ] ◄─────────────────────────────────── [ Redis Feed Cache ]

   
  Core Services :
    -> API Gateway / Load Balancer: Handles SSL termination, authentication, rate limiting, and routes requests to appropriate microservices.
    -> Post Service: Manages post creation, text persistence, and media upload signaling.
    -> User / Social Graph Service: Tracks user profiles, follow/unfollow actions, and social connections.
     -> Fanout Worker Service: Asynchronously updates news feeds of followers when a new post is published.
    -> Feed Service: Retrieves pre-computed or dynamically assembled feeds for requesting clients.
    -> Media Service / Object Storage: Stores media assets using S3/Cloud Storage, delivered globally via a Content Delivery Network (CDN).

3. Detailed Data Flow & Fanout Strategies :
   Write Path(Posting) :
     ->  User submits a post through the API Gateway.
     ->  Post Service writes the post metadata to the database and uploads media assets to Object Storage.
     ->  Upon successfully writing the post, the service publishes a PostCreated event to Apache Kafka.
     -> The Fanout Worker Service consumes the event and determines the distribution strategy based on author attributes.
     -> Fanout Strategies (Hybrid Approach)FeatureFanout-on-Write (Push Model)Fanout-on-Read (Pull Model)Hybrid Model (Recommended)MechanicPre-computes feeds and pushes post IDs to every follower's timeline cache.
     -> Fetches posts dynamically when a user requests their feed.
     -> Uses Push for normal users and Pull for high-follower accounts ("celebrities").
     -> ProsFast read performance ($O(1)$ lookup in Redis).
     -> Fast write performance ($O(1)$ post creation); low cache overhead.
     -> Balances read/write traffic; prevents fanout bottlenecks.
   
   Cons :
     -> High write amplification for users with large follower counts.
     -> Slow read response times ($O(N)$ fetching and sorting at query time).
     -> Added architectural complexity in the fanout layer.
   
   Implementation Logic :
     -> Standard Users ($< 10,\text{000}$ followers): Fanout-on-Write.
     -> The worker service fetches the author's follower list and writes the post_id into each follower’s Redis timeline list.
     -> Celebrity/High-Follower Accounts ($> 10,\text{000}$ followers): Fanout-on-Read.
     -> The post is saved in the database, but not pushed to follower caches.
     -> When a user opens their feed, their timeline is assembled by merging their push-based Redis cache with recent posts from celebrities they follow.

4. Database Schema & Data Storage :
    Data Stores Selection :
    Relational DB (PostgreSQL / MySQL with Sharding): Stores transactional data such as users, posts, and comments.
    NoSQL / Graph DB (Cassandra / Neo4j): Stores follower graph data for fast edge traversals.
    In-Memory Cache (Redis Cluster): Stores pre-computed feed timelines (stored as Redis Sorted Sets (ZSET) where key = user_id, score = timestamp, value = post_id).
 
5. Technical Considerations & Trade-Offs :
     Scalability & Availability :
       -> Database Sharding: Database tables are sharded using user_id to distribute read and write traffic evenly across DB clusters.
       -> Stateless Application Servers: All app instances are stateless and scale horizontally behind standard Cloud Load Balancers.
     Cache Management :
       -> Redis Eviction Policy: Configured to volatile-lru or restricted by feed depth.
       -> Timelines keep only the top $N$ post IDs (e.g., 500 most recent items) per active user. Inactive users' feeds are evicted from cache and re-assembled on demand upon login.
     Security :
       -> Authentication & Authorization: Secure JWT verification at API Gateway level.
       -> Rate Limiting: Token Bucket or Leaky Bucket algorithms implemented at Gateway to protect against spam and DDoS attacks.

    Monitoring & Reliability :
       -> Circuit Breakers: Implemented via resilience patterns to fall back to static or cached feeds if feed assembly services degrade.
       -> Distributed Tracing: Implemented with OpenTelemetry across microservices to identify bottlenecks in the write and read pipelines.
