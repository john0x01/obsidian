# Case Study: Social Feed

## Requirements and Scale

### Timeline / News Feed Requirements

### Scale Estimation (Users, Posts, Fan-Out)

## Fan-Out Strategy

### Pull (Fan-Out on Read) vs Push (Fan-Out on Write)

### Hybrid Fan-Out for Celebrities

## Feed Storage and Caching

### Feed Storage Structure

### Caching Strategies for Feeds

### Pagination of Timelines (Cursor-Based)

## Ranking

### Chronological vs Ranked Feeds

### Feed Ranking and Personalization

### Deduplication and De-ranking

## Content Propagation

### Deletion and Edit Propagation

### Media Embedding

### Real-Time Updates (Push, SSE, WebSocket)

## Privacy, Discovery, and Safety

### Privacy and Audience Filtering

### Hashtag and Search Integration

### Abuse and Moderation Pipeline

### Analytics and Engagement Tracking

## See Also
- [[caching]] — caching feeds
- [[sharding-strategies]] — sharding feed storage
- [[06-software-design/api-design/server-sent-events|Server-Sent Events]] — real-time feed updates
- [[scalability]] — fan-out scaling
