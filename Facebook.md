# Social Media Platform Architecture

## Overview
This document outlines the comprehensive architecture of a scalable social media platform, detailing the microservices, data flow, and infrastructure components required to support core functionalities like user management, content creation, social interactions, and real-time notifications.

## 🔐 1. User Onboarding / Login Flow

### UI Components
- User Onboarding/Login Flow interface

### Services
- **User Service**: Core authentication and user management
  - Validates user credentials
  - Stores new user data in User DB MySQL Cluster (Master + Slaves)
  - Manages user sessions/tokens (JWT or cookie-based)
  - Caches sessions and metadata in Redis Cluster for fast access
  - Returns session/auth tokens to frontend

### Data Storage
- **User DB MySQL Cluster**: Master-slave configuration for user data persistence
- **Redis Cluster**: Session caching and metadata storage

---

## 👥 2. Friendship / Graph Flow

### UI Components
- User Add Friend Flow interface

### Services
- **Graph Service**: Manages social connections
  - Updates User Graph DB MySQL Cluster with bidirectional relationships
  - Updates Redis Cluster for real-time friend graph queries
  - Publishes friend activity events to Kafka

### Data Storage
- **User Graph DB MySQL Cluster**: Social relationship data
- **Redis Cluster**: Real-time friend graph caching
- **Kafka**: Event streaming for friend activities

---

## 📝 3. Post Creation Flow

### UI Components
- Adding Post UI interface

### Services
- **Post Ingestion Service**: Content validation and processing
- **Asset Service**: Media handling
  - Recent assets → CDN (Cloudflare, Akamai)
  - Archived/original assets → AWS S3
- **Short URL Service**: Link shortening functionality
- **Post Service**: Post data management
- **Post Processor**: Content enrichment and metadata processing

### Data Flow
1. Post Ingestion Service validates content
2. Assets sent to Asset Service for storage/CDN distribution
3. Short URLs generated for links
4. Structured post data sent to Kafka
5. Post Service subscribes to Kafka topic
6. Posts stored in All Posts Cassandra Cluster
7. Post Processor enriches content and pushes to Timeline Service

### Storage
- **All Posts Cassandra Cluster**: Primary post storage
- **AWS S3**: Asset archival storage
- **CDN**: Recent asset distribution
- **Archival Cron**: Periodic storage optimization

---

## 📰 4. Timeline Generation & Serving

### Services
- **Timeline Service**: Feed generation and ranking
  - Pulls user's friends via Graph Service and Redis cache
  - Fetches relevant posts from Post Processor, Redis, or Cassandra
  - Assembles and ranks posts using recency, interactions, and personalization scores

### Caching Strategy
- **Redis Cluster**: Timeline feed caching for fast retrieval

### UI Components
- **Timeline Screen**: Fetches feed via Timeline Service

---

## 👤 5. Profile View Flow

### UI Components
- User Profile Screen

### Services Integration
- **User Service**: Profile data retrieval
- **Timeline Service**: User's personal posts (excluding friends' posts)
- **Asset Service**: Media content retrieval

---

## 🔍 6. Search Flow

### UI Components
- Search Screen interface

### Services
- **Search Service**: Query processing and routing
  - Checks Redis Cluster for popular/recent queries (cache hits)
  - Queries ElasticSearch Cluster for cache misses
- **Kafka Consumer**: Real-time content indexing
  - Reads post activity from Kafka
  - Indexes new content into ElasticSearch

### Data Storage
- **Redis Cluster**: Search query caching
- **ElasticSearch Cluster**: Full-text search indexing

---

## ❤️ 7. Like & Comment Flow

### UI Components
- Like/Comment UI interfaces

### Services
- **Like Service**: Like functionality management
- **Comment Service**: Comment functionality management
- Both services:
  - Store data in respective Cassandra Clusters
  - Use Redis for recent likes/comments on trending posts
  - Publish events to Kafka
  - Notify Timeline Service for live feed updates

### Data Storage
- **Cassandra Clusters**: Like and comment data storage
- **Redis**: Hot post interaction caching

---

## 📈 8. Activity Tracking, Trends, and Notifications

### Activity Tracking
- **Activity Tracker Service**: Comprehensive user action tracking
  - Tracks clicks, likes, posts, and other interactions
  - Sends activity logs to Kafka
  - Stores data in All Posts Cassandra Cluster for behavioral analytics
  - Forwards data to Hadoop Cluster for batch processing

### Streaming Jobs
- **Apache Spark Streaming Cluster**: Real-time data processing
  - Listens to Kafka streams
  - Generates trending topics and viral post identification
  - Creates user profiling data
  - Outputs to Trends UI and Redis Cluster

### Batch Jobs
- **Hadoop Cluster**: Heavy periodic processing
  - **User Profiling Job**: Builds comprehensive user behavior profiles and interest mapping
  - **Graph Weight Job**: Assigns friend influence scores and relationship strength metrics

### UI Components
- **Trends UI**: Displays trending content via Trends Service (Redis-backed)

---

## 🔔 9. Live Notifications via WebSockets

### Real-time Communication
- **Live User WebSocket (Notification) Service**
  - Maintains persistent WebSocket connections with user browsers/apps
  - Consumes Kafka events (likes, comments, friend requests)
  - Delivers real-time notifications:
    - "X liked your post"
    - "Y commented on your post"
    - "Z accepted your friend request"

---

## 📊 10. Analytics Pipeline

### Data Collection
- All service events pushed to Kafka for centralized processing

### Analytics Processing
- **Analytics Service**: Kafka topic consumption
  - Aggregates key metrics: DAU (Daily Active Users), retention rates, engagement metrics
  - Exports data to BI tools and dashboards

---

## 🧠 Supporting Technologies and Design Patterns

| Component | Purpose & Details |
|-----------|------------------|
| **Kafka** | Enables real-time, decoupled, distributed data pipelines across all system components |
| **Redis** | Heavily utilized for caching timelines, friend graphs, trending topics, and search queries |
| **Cassandra** | Write-optimized, horizontally scalable database ideal for timelines, likes, comments, and activity logs |
| **ElasticSearch** | Provides full-text indexing and advanced search capabilities |
| **Hadoop + Spark** | Combined platform for batch analytics and real-time stream processing |
| **Microservices Architecture** | Each functional domain (timeline, posts, users, search) is independently scalable and deployable |

## Architecture Benefits

### Scalability
- Horizontal scaling capabilities across all components
- Independent service scaling based on demand

### Performance
- Multi-layer caching strategy (Redis, CDN)
- Optimized data storage for different use cases

### Reliability
- Event-driven architecture ensures loose coupling
- Master-slave database configurations for high availability
- Distributed storage systems for fault tolerance

### Real-time Capabilities
- WebSocket connections for instant notifications
- Kafka streaming for real-time data processing
- Redis caching for sub-second response times
