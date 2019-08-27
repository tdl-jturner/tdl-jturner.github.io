+++
draft = false
date = 2019-08-26T14:19:54-07:00
title = "Designing Flutter"
+++

I've been thinking of a method to experiment with new technologies in a practical way beyond just Hello World. The goal would be to make a system that is complex enough to be valid but still simple enough to rapidly deploy in new languages and technologies. In addition, it would be broken in to microservices such that any individual method could be quickly swapped out and play well together. This way if I wanted to explore a new language or technology then I could simply rewrite a single API and see what happens. This application will be called Flutter and will be a simple Micro Blogging Service (aka Twitter clone).

## Use Cases

The basic use cases of Flutter.

1. Post a message to the system
2. Read all messages from a user
3. Follow another user
4. Read all messages from followed users
5. Receive notifications of messages posted by followed users
6. Search messages

## Architecture
{{<mermaid>}}
graph TD
    UI-->ServiceRegistry
    ServiceRegistry-->MessageService
    ServiceRegistry-->SearchService
    ServiceRegistry-->TimelineService
    ServiceRegistry-->FollowService
    ServiceRegistry-->UserService
    TimelineService-->MessageDAO
    SearchService-->MessageDAO    
    MessageService-->MessageDAO
    MessageService-->NotificationService
    UserService-->NotificationService
    UserService-->UserDAO
    FollowService-->FollowDAO
    MessageDAO-->DataStore
    UserDAO-->DataStore
    FollowDAO-->DataStore
{{< /mermaid >}}

The UI will go through a Service Registry which knows about each of the APIs from the given microservices. The Service layer will sit on top of the DAO layer which in turn interacts with the DataStore. Services should individually be able to be plugged in and out as necessary. The DAO will be more closely tied to the DataStore selection. Each layer will pass data as JSON representations for simplicity and consistency.

## Entities
The following will be the basic entities used throughout the system.

### User
| name                 | type      |
|----------------------|-----------|
| id                   | uuid      |
| username             | text      |
| email                | text      |
| enable_notifications | boolean |
| created_dttm         | timestamp |

### Message
| name         | type      |
|--------------|-----------|
| id           | uuid      |
| author       | uuid      |
| text         | text      |
| created_dttm | timestamp |

### Follow
| name         | type      |
|--------------|-----------|
| author       | uuid      |
| follower     | uuid      |

## What's Next
The initial POC for this will be done with Java and Cassandra as these are technologies I know well and can run with. Once the initial POC is up and going, I'll start looking at different products to piece in here and experiment. My next post will focus on building out these services.
