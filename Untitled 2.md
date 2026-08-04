Framework

Requirements
Core Entities - Nouns
API/Interface
High Level Design - Satisfy Functional Requirements(Need Not be scalable)
Deep Dive

Functional Requirements
Create Posts
Follow users
View feed

Non-Functional Requirements
CAP 
1. Eventual consistency (1 minute)
2. 500ms latency for posting and viewing
3. 2B DAU

Core Entities
User, Post

API
POST /posts
{
	content:
} -> 201

PUT /users/id/followers

GET /feed?pageSize={size}&cursor={cursor}
{
	posts: []
	 nextCursor: ""
}

event_window: null

