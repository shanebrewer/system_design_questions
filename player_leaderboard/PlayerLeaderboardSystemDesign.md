# Epic Games Player Leaderboard System Design

Shane Brewer (shane.brewer@gmail.com)

## Problem Statement

Build a minimal player-facing leaderboard for one of Epic's multiplayer online games for the top 100 scores. Support 4 different game types and daily, weekly, and all-time aggregations.

---

## Product Requirements

### Personas

Personas are ways to categorize and define our users. They provide attributes that help us to understand who they are, what skills they have, what they are trying to accomplish, and what problems they are trying to solve.

#### Persona: Average Player

This person plays the game as a hobby. They may attend school or have a job. They play the game for entertainment value.

#### Persona: Professional Streamer

This person plays the game and streams their play to an audience. They collect donations or may sell custom subscriptions or merchandise. This person treats their playing more like a business than just a hobby.

#### Persona: Professional Gamer

This person plays the game competitively. They enter competitions and attempt to win prize money.

### Functional Requirements

1. A user can view the leaderboard for a game, which includes the top 100 players for 4 play modes and daily, weekly and all-time aggregations.
1. A user can view their own ranks in each of the 4 games modes and daily, weekly, and all-time aggregations.
1. A user can search for another user's ranks based on their username.
1. Games will be played worldwide, requiring internationalization

### Non-functional Requirements

1. The system should be able to handle the load of up to 10 million concurrent players and averages 10,000 game matches end every minute
1. The system should be highly available (Assuming that we can sacrifice strong consistency in favor of eventual consistency), with a minimum availability of 99.95%.
1. The system should be highly scaleable, being able to handle more traffic linearly with adding additional hardware.
1. The system should maintain 99.999% persistence.
1. The system should throttle users reading the leaderboard at 1 transaction per second (TPS) to prevent the system from being overrun by a bad actor.
1. The system should only interact with authenticated clients.
1. The system only allows authenticated game servers to write game information to the database and leaderboard.

### Out of Scope Requirements

1. A user can share their ranks on social media networks.
1. A user receives notifications when they enter the top 100 in any game mode across any aggregation.

---

## Assumptions and Capacity Estimation

### Assumptions

1. We will be building this solution using a major cloud provider.
1. The game being played will have up to 10 million concurrent players.
1. On average, 10,000 game matches will end every minute.
1. A game score can be recorded using a 32-bit integer.
1. With an assumption that the game will have up to 10 million concurrent players, we will assume that we have a daily active user population of approximately 100 million players.
1. Of the 100 million daily active players, approximately 10 million of them will check the leaderboard once per day.
1. Each game will be played by an average of 10 players.
1. The different aggregation time spans will utilize the date and time that the game starts. For example, for a game to be considered in the leaderboard for the daily aggregation, the game needs to have started less 24 hours from DateTime.now.
1. The daily aggregation will roll score off the leaderboard on a minutely basis. For example, for a high score on the daily leaderboard played at 1:05:XX pm yesterday, it will fall off the leaderboard at 1:05 pm today. We will not consider seconds in the removal of a score from the leaderboard.
1. The weekly aggregation will be roll score out on a daily basis.
1. Each of the 4 Game Types are played equally. NOTE: This is not likely how it would be in reality, but we will assume this for this question. It would be preferred to utilize averages based on real-world metrics.

### Traffic

#### Write Traffic

Assuming that 10,000 game matches will end every minute, we need to record scores in the database for each player and determine if it is a top score across the 4 game types and aggregations. Assuming an equal distribution of those games ending across that minute, we can assume the following write TPS:

    10,000 games finishing per minute / 60 seconds = 167 TPS

#### Read Traffic

Assuming that 10 million players will check the leaderboard once per day, we can assume the following read TPS:

    10 million views per day / 24 hours / 60 minutes/ 60 seconds = 116 TPS

### Bandwidth

Based on our traffic assumptions, we can now make an assumption for our bandwidth expectations. More details about some of the assumptions made for sizes can be found in the Data Model section below.

#### Write Bandwidth

We can assume that a write into the database for a game score would take approximately:

    Game_ID: 8 bytes
    User_ID: 8 bytes * 10 players = 80 bytes
    Game Score: 4 bytes * 10 players = 40 bytes
    Region Played: 20 bytes
    Game Mode: 4 bytes
    Metadata: 20 bytes

For a total of approximately 172 bytes. If we round up to 200 bytes to account for some expansion buffer, we are looking at:

    167 TPS * 200 bytes per write = ~33 KB per second

#### Read Bandwidth

Assuming a read of the leaderboard would consist of the following size:

    Usernames: 50 bytes * 100 players * 4 game modes * 3 aggregations = 215,000 bytes (Assumes UTF-32 encoding, but could be reduced by only supporting UTF-8)
    Scores: 4 bytes * 100 players * 4 game modes * 3 aggregations = 4800 bytes

For a total of approximately 219,800 bytes. If we round up to 250 KB, we are approximately our read bandwidth at:

    250 KB * 116 TPS = 29 MB/sec

### Storage

For storage, each game played we will need to store the game score and details, which we estimated to be approximately 100 bytes per game per player. This means we need to store:

    100 bytes * 10 players per game * 10,000 games per minute * 60 minutes * 24 hours * 365 days = ~5.3 TB/year

There would be additional storage for data such as player information but that data will grow in relation to the number of players rather than the number of games played, which is significantly less.

---

## High-Level System Design

The following is an initial basic system design prior to any kind of optimization or scaling.

![High-level System Design Diagram](/docs/diagrams/out/high_level_system_design/high_level_system_design.png)

### Clients

The clients represent the consumers of the APIs that the system will be exposing. In this case, there could potentially be logic in the game to display the leaderboard, as well as a web client where a player could view the leaderboard on a webpage, or a companion mobile app. Additionally, a game server would be utilizing our write API to write game scores to the database.

### API Gateway

We would look to run an API Gateway to expose our APIs. This offers the following pros and cons:

Pros

* This provides some security as it allows us to expose our APIs while hiding our backend implementation.
* We could potentially cache some results on this server.
* This server could implement client throttling, helping to protect our servers from being overrun by a bad client.
* This provides us with the ability to scale servers behind the API Gateway without the clients knowing about the change.

Cons

* This introduces additional complexity to the design as well as additional servers to manage and maintain.
* The API Gateway can be a single point of failure without additional availability strategies (Discussed later)

### Application Servers

We would have 2 different web servers, 1 for our write APIs and 1 for our read APIs. Splitting these 2 servers apart allows us to scale each one independently, as our read and write traffic significantly differs from one another. Our API Gateway will route write APIs to the Write API service, while read APIs will be routed to the Read API service.

### Database

Finally, we will need a database to persist our data. I'm leaving the type of database used open here, and will discuss the tradeoffs in the [Database and Data Model](#database-and-data-model) section.

---

## API

Next, we will define the APIs that will be exposed by our system. Before we do that, we should ask ourselves, "What makes a good API?"

Generally speaking, well-defined APIs have the following characteristics:

* They make code readable. APIs should make it clear in code what is being done.
* They are consistent. Attributes or objects across APIs should be and react the same across every API.
* They are backward compatible. Changes to APIs should never break existing clients. Versioning APIs can help address this.
* Easy to learn. Semantics should be simple and clear.
* Easy to extend.
* Work backward from user use cases.

### Write API

Every game, we will need to record the player's score, as well as ensure that all aggregation calcuations are completed.

    SubmitGameResults(client_key, game_type, results{player_id, score}[], region, start_time, end_time)

Parameters:

    client_key (string): API key of the client.
    game_type (string): The game type that was played.
    results{player_id (64-bit integer), score (32-bit integer)}[]: An array of tuples, specifying the player_id and the corresponding score the player received in the game. Each player in the game has tuple in the array. 
    region (string): The server region that the game was played on. No specific requirement for this, but requirements could be extended to provide a leaderboard in each region. 
    game_start_time (DateTime): The start time of the game.
    game_end_time (DateTime): The end time of the game. 

Responses (Not exhaustive):

    200: Success
    400: Bad Request (Badly formed request)
    401: Unauthorized (If the client service isn't authenticated)
    403: Forbidden (To ensure only allowed clients can write game scores)
    500: Internal Service Error (Generic server error)

### Read API

Next will be the read API to obtain the leaderboard information. This includes all game types and all 3 aggregations.

    GetLeaderboard(client_key)

Responses

    200: Leaderboard Information
        Game_Type (string)[{Player Name (string), Score (32-bit Integer)}[100]][4]
    400: Bad Request (Badly formed request)
    401: Unauthorized (If the client service isn't authenticated)
    403: Forbidden (Available if we want to restrict who can view the leaderboard)
    500: Internal Service Error (Generic server error)

---

## Database and Data Model

Next up, we'll take a look at the data model for the data we will need.

![Data Model](/docs/diagrams/out/data_model/data_model.png)

### Player Table

This database entity stores information related to the Player, including a Player_ID unique primary key, personal information, and account metadata.

### Game Table

The Game entity stores information specific to a single game. Each game played would have an entry in the database, providing information about the game itself including when it started, ended, the game type and the region where the game was played (For managing multiple regional game servers).

### Game_Score Table

The Game_Score table stores a single score for a player from a game. Each entry has a Foreign Key relationship to a single Player and a single Game.

### Analysis

Now that we have an initial data model, let's look at how the database would be accessed to load the leaderboard. For daily and weekly aggregations, we would need to get all of the Game_IDs for games that have started in the past 24 hours and 7 days respectively, for a total of:

    10,000 games per minute * 60 minutes * 24 hours = 14,400,000 Games per Day
    14,400,000 Games Per Day * 7 Days = 100,800,000 Games per Week

For each of those games, we would need to get the scores for an average of 10 players per game and then determine the highest scores. This also doesn't even consider the highest-scores of all-time, which would require a complete database lookup. Given the huge number of hits to our database, this would be extremely inefficient, so we should look at alternative storage options to be able to access a leaderboard.

![Leaderboard Data Model](/docs/diagrams/out/leaderboard_data_model/leaderboard_data_model.png)

The Leaderboard Table would store the top 200 scores for each game type. We would have 1 table for the daily aggregation, one for the weekly aggregation, and one for all-time. Why 200? This will allow us to roll out a score from the daily or weekly aggregations once it becomes too old to be in that aggregation and have a score that can take its place without needing to query the long-term storage databases.

It's important to note that we will need to build and maintain this database utilizing application logic as part of web service somewhere in our system, most likely in our Write API service.

Regardless of this optimization, each time we are loading the leaderboard means that we are making 1200 calls (Top 100 scores * 4 Game Types * 3 Aggregations) database calls. To improve this, we can maintain a cache, which I will discuss in the [Caching](#caching) section.

### Database Selection

Now that we have an idea of the data model and access patterns, we should make a decision on the database we will utilize.

#### Relational and Non-Relational

##### ACID Compliance

Given that our data is not critical (e.g. Payment Transactions, Mission Critical), it isn't necessary that we have ACID compliance. This means we could go either Relational or Non-relational.

Best Option: Either

##### Strict Schema

As development around a game is constantly in flux, it may make sense to allow for a flexible scheme. That way, additional features could easily be added without needing complicated database operations.

Best Option: Non-relational

##### Querying

Relational databases provide SQL for defining and manipulating data. As we don't really have a need for complex joins or queries, either option will work for us.

Best Option: Either

##### Scalability

Relational Databases are vertically scalable, while non-relational databases are horizontally scalable.

Best Option: Non-relational

#### Decision

Given this data, we will choose a non-relational database as it appears to be the best option for our use cases. As for the type of Non-relational database a Key-Value pair database, which provides us with O(1) lookups, with Document capabilities, which allows us to expand the schema as needed, would give us excellent flexibility, performance, and scalability. A solution such as [AWS DynamoDB](https://aws.amazon.com/dynamodb/) would give us these capabilities.

### Persistance

While most cloud-based database services offer options for on-demand backup, setting up multiple database replicas across different regions provides an excellent disaster-recovery option to ensure our data is persisted long-term.

---

## Scaling

Next up, we want to look at ways to scale our initial system. It's important to note that we wouldn't normall just scale a system, but would look to load test our system, obtain profiling information, and identify bottlenecks and look to address those bottlenecks to get our system to handle expected traffic.

### DNS

Adding DNS means that we can swap out an API Gateway with another instance in cases of large outages (Regions), or if we want to move to a different version (i.e. Azure) without needing to make changes to our clients.

![System Design V2](/docs/diagrams/out/high_level_system_design_v2/high_level_system_design_v2.png)

### Content Delivery Network (CDN)

While there isn't an immediate need for this based on our use cases, it is logical that we could add features like custom avatars or images to Player's profiles that would show up on the leaderboard. For use cases like that, we could have a Push CDN where we would push images like this to and then have the clients call the CDN directly to get these images.

![System Design V3](/docs/diagrams/out/high_level_system_design_v3/high_level_system_design_v3.png)

#### CDN Disadvantages

1. Adding a Push CDN adds some complexity to the development process as images need to be pushed to the CDN.
1. Client code needs to utilize URIs that point to the CDN.

### Horizontal Scaling

We can add horizontal scaling to our system by adding multiple hosts that are fronted by a Load Balancer (LB). The LB then distributes requests to application servers based on the configured distribution algorithm, while also monitoring the health of the services and taking servers with bad health out of the pool. This will help improve reliability as bad servers are removed from the pool of servers receiving requests, while also allows us to scale up with additional hosts if traffic increases.

![System Design V4](/docs/diagrams/out/high_level_system_design_v4/high_level_system_design_v4.png)

#### Horizontal Scaling Disadvantages

1. LBs can be a single point of failure. This can be mitigated with different availability patterns like running Active-Passive or Active-Active LBs. However, both of these patterns add additional complexity and cost.

### Caching

We can add an in-memory cache for the leaderboard so that we never need to make a database call when performing a GET for the leaderboard. We will need to populate and maintain this cache, but doing so will significantly improve performance while removing traffic from our databases.

#### Cache Size

To store the entire leaderboard in memory, we would need the follow space:

    4 Game Types * 3 Aggregations * 200 Scores = 2400 entries
    Player Username (string): ~20 bytes
    Score (32-bit Integer): 4 bytes
    Game Type (String): ~10 bytes
    Game Start DateTime: 8 bytes
    Total memory per entry: 42 bytes
    Total Memory Size: 42 bytes * 2400 entries = 100,800 bytes 

#### Cache Type

For our cache, we will use a Write-Aside cache type. This will enable reads of the leadboard table to hit the cache, while also ensuring we persist the data in a database which will allow us to populate the cache incase we suffer a crash. The Leaderboard Table could also act as a fallback option for reads in the case that the cache is not available.

### Additional Use Cases

#### Expirying a High Score

For the daily and weekly aggregation, we need to remove a high score from that table when it reaches the time limit and replace it with the next highest score. To do that, we need to track when a score on a leaderboard was for a game  either more than a daily or a weekly ago. To do this, we can maintain another service called "Score Expiry Service" which will run cron jobs that will notify us when a score has expired. This will kick off the logic to remove the score from the leaderboard and replace it with the next highest score (101st highest score in this case). This ensures that we have the top highest scores for both the daily and weekly aggregations.

### Final System Design

![System Design V5](/docs/diagrams/out/high_level_system_design_v5/high_level_system_design_v5.png)

---

## Main Use Cases

The following section shows the flow across our services for the main use cases that we will see.

### Write Game Score

When a game finishes, the client will look to store the game results. During this phase, we will also look to update the leaderboard cache and table if a score is in the top 200 scores for the game types across any of the 3 aggregations (Daily, Weekly All-time).

![Write Sequence Diagram](/docs/diagrams/out/write_game_data_sequence_diagram/Submit_Game_Data.png)

In this design, the Write API service is responsible for updating the leaderboard cache, and all databases.

### Read Leaderboard

When a client wants to get the leaderboard, the following call pattern occurs:

![Read Sequence Diagram](/docs/diagrams/out/read_leaderboard_sequence_diagram/Read_Leaderboard.png)

### Populate Cache

In the case that we've suffered a crash on our cache system, we can utlize the Read API Service to pull the leaderboard from the Leaderboard Database while the cache service is down (At a performance cost), and then once the Cache service is back up and running again, use the Read API Service to populate the cache.

![Populate Cache Sequence Diagram](/docs/diagrams/out/populate_cache_sequence_diagram/Populate_Cache.png)

### Expire Score

Finally, when a score "expires", or was played longer in the past than the aggregation period (e.g. 24 hours or 7 days), then we need to update the leaderboard cache and database to reflect this change.

![Populate Cache Sequence Diagram](/docs/diagrams/out/expire_score_sequence_diagram/Expire_Score.png)

---

## Operations

Now that we have a service running in Production, we need to monitor the health of that system. That includes ensures that we are paying attention to the following metrics:

### Performance

To ensure requests are receiving a response in a timely manner, we should monitor P99 latency across all of our APIs. Ideally this would be done at the Client, API Gateway, and Application Server levels, to allow us to determine where latency is being added in the call hierarchy.

### Response Quality

For every request that we receive, we would like to get a count of every response codes given back to the client. This includes % of total requests, so that we can determine the porportion of traffic that is receiving client-based errors (4XXs) or server-based errors (5XXs). Monitoring these metrics would allow us to detect issues on either side and set threasholds to notify us of major issues. 

### Host Metrics

For our application servers, we will want to monitor metrics including:

* CPU Utilization: So that we know when we are maxing out host CPU resources.
* Memory Utilization: So that we know when we are close to exhausting a host's available memory
* Disk Space Utilization: To ensure we have enough disk space for the host to continue to run properly.
* OS Threads: To monitor how many open OS threads we have to ensure we don't start to thrash or starve a thread.

### Canaries

In addition to all of this, running canaries in Production to ensure the availability and functional correctness provides additional protection and speeds up the detection of issues.

---

## Tech Stack

For our tech stack, I'd make use of the following AWS services:

### DNS -> Route 53

Route53 provides custom DNS routing capabilities. This allows us to swap out our API Gateway for another instance if needed, perform A/B Testing if needed, and enable routing to multiple regions if needed.

### CDN -> CloudFront

In the instances that would we utilize a CDN (e.g. User Avatars), I would make use of AWS CloudFront for storing images. 

### API Gateway -> AWS API Gateway

Our API Gateway would use AWS API Gateway. This provides us with features including utilizing OpenAPI specification as input for defining the interface, client-based throttling to help prevent a bad actor from making our system unavailable, provides excellent metrics and integration AWS CloudWatch, automates the process of generating an SDK for our client applications, and integrates with AWS for Authentication and Authorization.

### Application Services -> AWS Lambda or EC2

For our application services, we'd look at AWS Lambda or EC2. While Lambda provides us with a serverless option that reduces our operational burden, it also comes with a reduction in control, which can make things like successfully adapting to variable traffic difficult. EC2 provides with the ability to customize how we scale our hosts, but at the cost of operational overhead. Picking one or the other would be ideal if we could view the traffic patterns to determine if we will see extremely volatile spikes or not. However, given that we are fronted by an API Gateway, it would be possible to swap out one option for another without impacting the clients, but it would require additional engineering effort to do. 

### Distributed Cache -> ElastiCache

For our Leaderboard cache, we could make use of AWS ElastiCache, which provides both Reid and Memcached engines. ElastiCache provides us with sub-millisecond latency that will make loading and updating our leaderboard high-performant.

### Metrics, Alarms, and Dashboards -> CloudWatch

In order to monitor the health of our services and to detect issues we can utilize AWS CloudWatch. CloudWatch integrates with other AWS services to easy enable publishing of operational metrics and build graphs, dashboards, and alarms.

### Infrastucture -> CloudFormation/Cloud Development Kit

For defining our infrastructure setup, we can utilize AWS Cloud Development Kit (CDK) for defining our infrastructure as code. CDK allows us to model infrastructure using TypeScript, Python, Java, and .Net allowing engineers to setup and make changes in infrastructure in their IDE. CDK then outputs CloudFormation files, which gives us repeatable deployments, rollback procedures, and drift detection between configuration files and actual infrastructure.

### Auditing -> CloudTrail

In order to track changes made to our system, we'll want to enable CloudTrail to logs for actions made within our AWS account. This will keep a history of changes that were made, ensuring that we can back out inadvertent or malicious changes to our account.

### Programming Language

If we decide that we want to make use of AWS Lambda for our application services, we should consider that certain interpreted languages (e.g. Python, Node.js) will have a faster initial invocation when compared with compiled languages (Java, .Net), but will not reach the same level of performance. Given that we aren't doing a heavy amount of computation with this system, a programming language like Python offers rich community support and libraries, is currently popular amount engineers (See https://insights.stackoverflow.com/survey/2020), and fast initial invocations. This comes at the cost of dynamic typing, which can delay detection of errors to run-time, some performance impact, and some history in enterprise institutions.