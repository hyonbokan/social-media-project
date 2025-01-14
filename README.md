# social-media-project

```scss
    ┌────────────────────────┐
    │      Mobile App       │
    │ (iOS/Android Client)  │
    └─────────┬─────────────┘
              │  REST/GraphQL
              ▼
    ┌────────────────────────┐
    │   API Gateway / BFF    │
    │ (Backend For Frontend) │
    └─────────┬─────────────┘
              ▼
 ┌───────────────────────┐   ┌───────────────────────┐
 │   Auth Service        │   │    User Service       │
 │ (JWT/OAuth2, Profile) │   │  (Profiles, Followers)│
 └───────────────────────┘   └───────────────────────┘
              ▼                     ▼
 ┌───────────────────────┐   ┌───────────────────────┐
 │  Post/Channel Service │   │   Messaging Service   │
 │ (Images, Channels)    │   │ (Direct Messages)     │
 └───────────────────────┘   └───────────────────────┘
              ▼                     ▼
         ┌────────────────────────────────┐
         │      Notification Service     │
         │  (Push, In-App, Email)       │
         └────────────────────────────────┘

   ┌──────────────────────────────────────┐
   │ Database Layer                      │
   │(PostgreSQL OR MongoDB + Redis etc.)│
   └──────────────────────────────────────┘

```

Key points:

- A mobile app (iOS/Android) communicates with an API Gateway or Backend For Frontend (BFF)—a single entry point for all client requests.
- Individual microservices handle distinct domains:
  - **Auth Service**: Sign up, login, JWT/OAuth2 tokens.
  - **User Service**: Manages user profiles, follow relationships, recommended moms to follow, etc.
  - **Post/Channel Service**: Creates posts (text/image), organizes them into channels (preemies, preschools, etc.).
  - **Messaging Service**: For direct messages or group chats.
  - **Notification Service**: Sends push/in-app/email notifications for new messages, likes, replies, etc.
- In a smaller project or MVP, you may combine some of these into fewer services, but logically decoupling them ensures future features can be added more easily.

---

## Core Components & Responsibilities

### 1. API Gateway / BFF
- **Purpose**: Provide a single endpoint for the mobile app. Handles request routing to underlying microservices.
- **Implementation**: Could be a Node.js or Spring Boot gateway (e.g., with Spring Cloud Gateway) or an NGINX/Kong API gateway.
- **Benefits**:
  - Abstracts away internal service topology from the mobile client.
  - Easy to enforce cross-cutting concerns (authentication, rate limiting, logging).

### 2. Auth Service
- **Responsibilities**:
  - Account creation (email/password, possibly OAuth for sign-in with Google/FB/Apple).
  - Token generation (JWT).
  - Password resets, multi-factor authentication.
- **Data**:
  - Typically minimal user info: username, hashed password, email.
- **Tech**:
  - Spring Security or Keycloak (Java), or Auth0 as a SaaS solution.
- **Scalability**:
  - Generally stateless if using JWT.
  - Could store session blacklists in Redis if you need forced token invalidation.

### 3. User Service
- **Responsibilities**:
  - User profiles (bio, profile picture, location, child’s age, etc.).
  - Following/followers relationships or a friend system.
  - Possibly store a user’s “tags” or “channels” of interest.
- **Data**:
  - Rows for each user, references to their profile.
  - Follower/following relationships (stored in a separate table or a graph store if complex).
- **Scalability**:
  - For large-scale, use a cache (e.g., Redis) for quick lookups of user profiles.
  - Partition or replicate data if user base grows large.

### 4. Post/Channel Service
- **Responsibilities**:
  - Posts: text or image-based. Optionally short videos.
  - Channels/Topics: For example, “preemies,” “preschools,” “miscarriages,” etc. A post can belong to one or multiple channels.
  - Likes and comments (if you want typical social features).
- **Image Uploads**:
  - Use Object Storage (e.g., AWS S3, GCP Cloud Storage) for storing images.
  - Store the image URL or path in your database.
- **Scalability**:
  - High read/write volume for posts.
  - Implement pagination or infinite scrolling.
  - Store post metadata in a relational or NoSQL DB, depending on queries.

### 5. Messaging Service
- **Responsibilities**:
  - Direct messages between users. Possibly group chats as well.
  - Real-time or near-real-time using WebSockets (Socket.IO or Spring WebFlux).
- **Scalability**:
  - For real-time messaging, consider a Pub/Sub or broker approach (e.g., Kafka, RabbitMQ) or use a dedicated chat service.
  - Store chat history in a NoSQL store for fast writes.

### 6. Notification Service
- **Responsibilities**:
  - Push notifications for new DMs, likes, replies, or channel updates.
  - In-app notifications (badge counts, ephemeral alerts).
  - Email notifications for offline or periodic summaries.
- **Implementation**:
  - Use a queue or event-driven architecture (e.g., each service emits events to a message bus—“NewMessageEvent,” “NewLikeEvent”).
  - Notification Service listens and triggers push/email accordingly.
- **Scalability**:
  - Keep it asynchronous and stateless to handle a growing user base.

---

## Data Storage Decisions

### Relational (PostgreSQL / MySQL)
- Good for structured schemas, complex joins, or ACID transactions.
- Examples:
  - user → post → comment → likes.
- Good for reporting or moderate complexity queries.
- Scalability:
  - Handled via sharding or replication, but it’s more involved.

### Non-Relational (MongoDB / Cassandra)
- Good for document-like data (e.g., JSON posts with flexible structures).
- MongoDB is simpler for storing unstructured or semi-structured data.
- High write throughput or very large volumes of data may scale more smoothly.
- Challenges:
  - NoSQL complicates relationships like a “followers” graph.
  - Need to implement references manually or embed data.

**Recommendation**:
- Start with PostgreSQL for structured user relationships and channel membership.
- Use S3 or equivalent for storing images.

---

## Scalability & Feature Extensibility

### Microservice or Modular Monolith
- Microservices let you scale each domain independently.
- A modular monolith might be simpler at first but should be structured in modules for easier future microservice migration.

### Event-Driven
- Publish events (e.g., “UserPosted,” “UserMessaged”) to a message bus (Kafka, RabbitMQ).
- Decouples services, enabling easier feature additions (e.g., analytics).

### API Gateways & BFF
- Reduce round trips for mobile apps.
- Add new endpoints in the BFF for new features (e.g., parenting events or kid-friendly places).

### Caching
- Use Redis to cache frequently accessed data like user profiles or popular channel posts.
- Helps with read-heavy loads.

### Load Balancing & Autoscaling
- Deploy microservices in containers (Docker + Kubernetes).
- Scale out messaging or post services during usage spikes.

---

## Workflow Example

### 1. User posts in “Preschools” channel
- Mobile calls `POST /posts` (with text, optional image).
- Post/Channel Service handles it, stores metadata in DB, image in object storage.
- Publishes “NewPostEvent” to message bus.

### 2. Notification for followers
- Notification Service listens to “NewPostEvent.”
- Fetches post detail and user’s followers.
- Sends push notifications to relevant followers.

### 3. Direct message flow
- User A → `POST /messages` with content, recipient.
- Messaging Service saves to DB, triggers a “NewDirectMessageEvent.”
- Notification Service sends push notification to User B.

---

## Conclusion
A modular or microservice architecture ensures each domain—Auth, User Profiles, Posts/Channels, Messaging, and Notifications—can evolve independently.

- Use **PostgreSQL** for structured relationships and queries.
- Store images in an object storage like **S3**.
- Consider NoSQL if anticipating extremely large, unstructured data.

By combining API Gateway, domain services, event-driven notifications, and robust storage, you’ll build a scalable social app ready for future enhancements.
