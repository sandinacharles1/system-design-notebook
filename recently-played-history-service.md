# Design a "Recently Played" History Service

Track the last 50 songs played per user with low write latency and fast reads.

## Functional Requirements
* Be able to see last 50 songs in order
* Be able to delete a song
* No duplicate songs allowed
* You can look at last 50 while actively listening to a song
* Restricted to only 50
* Add a song to recently played after 30s
* Hard delete the song

## Nonfunctional Requirements
* Low Latency
* Fast Reads (caching?)
* Eventual consistency is tolerable

## API Endpoint Design
* **Rules:** URL Looks like `Collection / Item / Sub-collection`
  * `POST /users/{userID}/recently-played` (receives JSON payload)
  * `GET /users/{userID}/recently-played`
  * `DELETE /users/{userID}/recently-played/{trackID}`

## Rough Database Schema
* **Recent Listens Table:** `userID | songID | timestamp`
* `songID` information gets rehydrated from songs table's metadata
* We use a timestamp because we want to be able to read the information from decreasing order of time

### Database Types
* **NoSQL:** Their document will be automatically received in order, and since the structure of some NoSQL databases allow consistent item numbers and order like wide-column stores like Apache Cassandra or DynamoDB where they have a partition key (categorizes) (`userID`) and sorting key (sorts, `timestamp`)
  * We can partition the documents by `userID`, and we can sort it by `timestamp`.
  * **ISSUE:** If we have `timestamp` as a sorting key, then duplicates in track ID may occur because timestamp isn't unique. In the application/server level, we could deduplicate with logic checks. OR we could use `trackID` as a key and Local Secondary Indexes to sort?
* **SQL:** Consistent ACID transactions.
  * **Trade-offs:** Receiving the songs in order for a person requires a full scan. However, for the instance of a recently played service, there's les conflict that might require the need for SQLs safety and consistency, but rather we'd value speed and sortedness.
* **Cache:** Redis can auto-sort and trim, with `ZSortedSet`, although in case of network lag we can fall back to timestamp).

## System Operations & Architecture

### POSTING a new song
* We could directly have each listen initiate an HTTP request to add to the most recently played.
* We could have an endpoint for listening to a song that sends it to a message broker like Apache Kafka (because there are other tools that may need this information too, like the machine learning models). So, we have one endpoint listen and it sends the info to a pub/sub. Since Apache Kafka is persistent, it removes one less worry :-)

### SAVING a new song
* Since theres estimated thousands of listens per second, that would throttle our database.
* **Batching Requests:** Removes strain on our database by having a worker put multiple requests in one.
  * **The trade-off:** When we batch requests, it will take a certain period of time, so it will take a while for the users information to be updated. So, we can fix that with the following:
* **Caching:** We could keep a key-value document storage like Redis to store user's past 50 songs. Although, this will not be our source of truth since Redis is not persistent. This will aid in making requests faster, and having people access their information before a batch worker sends everything into the database

### UPDATING
* **(CHOSEN ONE) SSE:** One way (what we need). Long lived http connection to any server. can just update when necessary. its simple and always reconnects automatically, so i choose this option. It's great for getting updates across devices
* **Long Polling:** The network overhead staying up can be a waste of network, and if someone skips multiple songs, it will keep closing and reopening with every request with new headers (bunch of extra metadata), every time.
* **Websockets:** Requires more complexity, more than necessary for simple updates. It requires sticky sessions because of the TCP connection it started in one server. Too much. Plus, its best for fast heavy two-way communication like chats / gaming. Yeah it requires less header byte size because of its consistent connection, but if that connection drops thats a pain to implement for something simple.
* **CHOSEN ONE (Optimistic UI updates):** for a single device, Optimistic UI updates on the client (adding the song to the local UI state immediately at the 30-second mark) avoids holding millions of open HTTP connections on the servers, although since its local it won't know to update other devices concurrently, although if the other device runs a GET request they'll see the song (it was saved to the database)

### DELETING a new song
* We can also batch the request. However, since duplicates are not allowed, we need to search by timestamp.

### GETTING the top 50
* Easy retrieval in order from Redis. However, if redis doesn't have it, we can rehydrate it from the database
* **Avoiding N + 1 Queries:** One of our columns (songs) require rehydration from a second table. We DON'T want to go through EVERY SINGLE SONG and rehydrate them separately in different queries. So, we will first do 1 query to get all the songs, then another query that batches all the information for every single song in one. We reduce it from N + 1 to only 2.
