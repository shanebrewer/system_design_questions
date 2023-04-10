## Problem Statement

Design Google Search.

---

## Product Requirements

### Personas

A "Persona" is a collection of defining attributes that classify a population of users. They help us understand who is going to use the system, what knowledge they have, what problem they are trying to solve, and how they going to interact with the system.

#### Persona: Information Searcher

- This user has particular questions that they are looking to search the internet for answers to.
- Their technical knowledge ranges simplistic knowledge of how to navigate the web, to users with extensive techncial knowledge.
- 

#### Persona: Information Publisher

- This user publishes information to the internet either on their own on via a 3rd party tool (e.g. Wordpress)
- This user would like their webpage to show up in the results of applicable searches by "Information Searchers"

### Functional Requirements

1. As an information searcher, I can go to a web page and type in a search query and get back a list of applicable web pages, ordered by relevance to my search query.
1. As an information publisher, information I publish to the web will show up in information searchers queries when the information provided is relevant to the search query. This is done automatically, and without any action on the publisher's behalf.
1. As an information publisher, existing web pages that I modify will be updated in relevant searches.

### Non-functional Requirements

1. Time to show up in web searchers
1. Availability - Our search web page should be highly available.
1. Performance - User should get hot-search results in seconds. Edge cases can take longer. 
1. Eventual consistency
1. High concurrency
1. Robust - Handle poorly formatted HTML, unresponsive servers, etc.
1. Throttled - Shouldn't issue too many requests to the same server.

### Out of Scope Requirements

1. Public-facing API
1. Searching for images, audio, or videos.

---

## Assumptions and Capacity Estimation

### Assumptions

- 2 billion daily active users
- 100 billion web pages
- 1 billion new web pages every month
- Each web page is on average approximately 1 MB in size

### Traffic



#### Write Traffic



#### Read Traffic

2 billion daily active users
Average 5 searches per day
Total Searches = 2 billion DAUs * 5 average searches = 10 billion searches per day
10 billion searches per day / 24 hours / 60 minutes / 60 seconds = ~116,000 searches per second

### Bandwidth

#### Write Bandwidth

#### Read Bandwidth

### Storage

---

## High-Level System Design

The following is an initial basic system design prior to any kind of optimization or scaling.

![High-level System Design Diagram](/docs/diagrams/out/high_level_system_design/high_level_system_design.png)

### Clients

### API Gateway

### Application Servers

### Database

---

## API


---

## Database and Data Model

### Database Selection

#### Database Decision

---

## Scaling

### DNS

### Content Delivery Network (CDN)

### Horizontal Scaling

### Caching

#### Cache Size

#### Cache Type

---

## Main Use Cases

### Use Case 1: 

---

## Operations

### Performance

### Response Quality

### Host Metrics

---

## Tech Stack

---

## Links
