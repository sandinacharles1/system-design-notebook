# Design a Playlist Management Service

Support creating, deleting, and reordering tracks in a playlist. Focus on relational vs. document schema design and handling playlist ordering efficiently without rewriting the whole list on every move.

## Known Constraints (SCOPE)
* Create playlist
* Delete Playlist
* IMPROTANT: Reordering playlist
* IMPORTANT: playlist ordering

## Learned Constraints
* Multiple Users can edit

---

## Opening Playlist
* User Search 
* Search gets routed with DNS 
* Playlist Image retrieval with CDN
* API Gateway & Load Balancing
* Security (rate limiting, authorization, JWT)
* Routing to a regional backend server
* Retrieve tracks and version from local cache 
  * if miss: Receive some tracks from server cache
  * If cache miss, check database 

---

## Storing Playlist

### Values
* Database level serialization for concurrent writes
* Normalization for reduction of redundancy (Trade-off: Higher Latency reads)
* However, since tables are distributed, we need a way to be able to join information quickly

### Database Types
* **SQL:**
  * Perks: Built in ACID Compliance, Database Row locking, Best for data with specific structure, checking information across playlists
  * Trade-offs: slower reads compared to NoSQL
* **NoSQL (Document Store):**
  * Perks: Best for data without a specific structure, really fast since everything is consolidated in a document
  * Trade-offs: does not have built-in database locking (although we could use optimistic concurrency control rather than pessimistic concurrency control, but this misses the protection across multiple normalized databases)

### Choice
SQL since we can easily make changes across multiple playlists, and the ability to reduce concurrency.

### SCHEMA (all rehydrated from eachother via playlist ID)
* **Playlist Metadata:** Playlist ID | Title | Owner ID | Collaborative | Version
* **Playlist Tracks:** Playlist ID | Track ID | Owner | Location
* **Playlist Collaborators:** Playlist ID | Collaborator ID

---

## Editing Tracks

### Edit Conflicts
* Serialize operations at database level
* **Optimistic Concurrency Control (Assumes conflicts are rare):** We utilize version control. When the first item edits the database, we update the playlist's version, now the second one has an outdated version. However, it has to retry
  * Human Retry: They can revise the changes
  * Automated Retry: It happens anyway
  * **SCALING For high-traffic collaborative playlists:** Edits route through a Partitioned Queue (Kafka) keyed by playlist_id so whenever edits are on a playlist they go one at a time
* **Pessimistic Locking (Assumes conflict is frequent):** We lock the database row, so it has to wait, but guarentees high consistency. If the first person to go has slow network, it may be slow for everyone else despite the person's edit maybe not even going through.

### MY CHOICE AND WHY
I choose optimistic concurrency control with automated retry, in a playlist, two people editing at the same time is mainly infrequent. Pessimistic locking, they can't even edit until the other person is done, which can be a negative user experience, but if they're able to edit at the same time then the OCC manages it, that may be more effective.

### Additional info
Idempotency key to stop an edit from going through multiple times

---

## Reordering Tracks

### Conflict Management
* Edit Conflicts: Optimistic Concurrency Control
* Fractional Indexing to reduce rewriting the whole list
  * We store the song and its location in the playlist in numbers , then return in in sorted order
  * However, numbers may have a precision point that may get too long, so we could use something like letters (lexicographical string indexing (like Jira's LexoRank, using strings like "a", "b", "an"). You get the letters between letters, then another middle letter.

---

## Showing Track Updates (Collaboration)

### Server-Sent events
Keeping a single, standard HTTP connection open (instead of closing it) so the server can continuously push text-based updates to the client in one direction. We can use a pub/sub (publish, subscriber) to publish events in a message broker (routes to the right people. not a message queue) and sent to the subscriber (client) by pushing the notification to all servers
* **Pros:** Automatically reconnects when dropped since youre always open
* **Cons:** One way

### Long Polling
The client sends a normal HTTP request and the server holds it open until new data is available or a timeout occurs.
* **Pros:** Stateless since its like a regular http request. Reconnects automatically since each request initiates a long polling
* **Cons:** If an event doesn't occur, its a waste. One way. You have to continuously make http requests with bigger headers

### Websockets
The client sends an HTTP Upgrade request to the server then the connection switches to the WebSocket protocol (ws:// or wss://) and stays open indefinitely. Avoids regular http requests
* **Pros:** Two-way, lowest latency because of the small headers (less metadata), can support binary data (audio)
* **Cons:** Doesn't automatically reconnect. Amongst two people, their changes need to go to a server and you need to be on that server two so the playlist needs to be a sticky session and manage state. Manual health checks since it doesn't automattcally.

### CHOICE
SSE's. I'm assuming changes aren't so quickly frequent, and we don't want to manage state with websockets because we dont necessarily need that quick back and forth, rather, we'd want the server to only show the edits when they come. Plus, if the connection drops, it automatically retrieves the information
