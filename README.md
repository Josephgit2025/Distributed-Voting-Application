# Distributed Voting Application with Docker

A distributed and scalable voting application using a microservices architecture with Docker and Docker Compose. The application demonstrates the principles of asynchronous communication between services via Redis, with data persistence in PostgreSQL.

## 📋 Project Overview

This application implements a simple yet robust voting system with an architecture composed of three distinct services:

- **Poll** : Web interface for submitting votes
- **Worker** : Asynchronous vote processing
- **Result** : Real-time results display

### Architecture

```
┌─────────────┐          ┌──────────────┐
│   Browser   │          │   Browser    │
└──────┬──────┘          └──────┬───────┘
       │                        │
       │ HTTP                   │ HTTP/WebSocket
       │                        │
    ┌──▼──────────────────────┐ │    ┌──────────────────────┐
    │   Poll Service (5000)   │─┼────│  Result Service      │
    │  (Python/Flask)         │ │    │  (Node.js/Express)   │
    │                         │ │    │      (5001)          │
    │  - Voter Interface      │ │    │                      │
    │  - Vote Submission      │ │    │  - Display Results   │
    │  - Cookie-based ID      │ │    │  - Real-time Updates │
    └────────┬────────────────┘ │    │     (Socket.io)      │
             │                  │    └──────────┬───────────┘
             │ Redis Queue      │               │
             │ (FIFO)           │               │ PostgreSQL Query
             │                  └───────┬───────┘
             │                          │
       ┌─────▼──────────────────────┬──▼──────────────┐
       │      Redis (6379)          │   PostgreSQL    │
       │   - Vote Queue             │    (5432)       │
       │   - Message Passing        │   - Vote Store  │
       │                            │   - Queries     │
       └─────┬──────────────────────┴─────────────────┘
             │
             │ Consume Queue
             │
    ┌────────▼────────────────────┐
    │  Worker Service             │
    │  (Java/Maven)               │
    │                             │
    │  - Listen to Vote Queue     │
    │  - Process Votes            │
    │  - Insert/Update in DB      │
    └─────────────────────────────┘
```

## 🚀 Quick Start

### Prerequisites

- Docker
- Docker Compose
- (Optional) Python, Node.js, Java for local development

### Installation and Launch

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd T-DOP-501-NAN_joseph-marie-bile
   ```

2. **Configure environment variables**
   ```bash
   cp .env.example .env  # If the file exists
   ```
   
   Or create an `.env` file with:
   ```env
   REDIS_HOST=redis
   POSTGRES_HOST=db
   POSTGRES_PORT=5432
   POSTGRES_USER=postgres
   POSTGRES_PASSWORD=postgres
   POSTGRES_DB=votes
   ```

3. **Start the application**
   ```bash
   docker-compose up --build
   ```

4. **Access the application**
   - **Voting Interface** : http://localhost:5000
   - **Results** : http://localhost:5001

## 📦 Services Architecture

### 1. Poll Service (Python/Flask)

**Port** : 5000

**Responsibilities** :
- Voting web interface
- Unique voter ID generation (cookie-based)
- Vote submission to Redis queue
- Default options: Ansible, Chef, Puppet, SaltStack

**Main files** :
- `poll/app.py` : Flask application
- `poll/templates/index.html` : HTML interface
- `poll/static/stylesheets/style.css` : CSS styles

**Environment variables** :
- `OPTION_A`, `OPTION_B`, `OPTION_C`, `OPTION_D` : Customizable voting options
- `REDIS_HOST` : Redis host

**Flow** :
1. User visits the voting interface
2. A `voter_id` is generated and stored in a cookie
3. User selects an option and submits
4. Vote is serialized as JSON and sent to Redis via `rpush('votes', data)`

### 2. Worker Service (Java/Maven)

**Responsibilities** :
- Asynchronous vote processing
- Redis queue consumption
- Storage/Update in PostgreSQL
- Automatic reconnection on failure

**Main files** :
- `worker/src/main/java/worker/Worker.java` : Worker application
- `worker/pom.xml` : Maven configuration

**Maven Dependencies** :
- `org.json:json` : JSON parsing
- `redis.clients:jedis` : Redis client
- `org.postgresql:postgresql` : PostgreSQL driver

**Environment variables** :
- `REDIS_HOST` : Redis host
- `POSTGRES_HOST`, `POSTGRES_PORT`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`

**Flow** :
1. Connects to Redis and PostgreSQL
2. Listens to `votes` queue via `blpop()` (blocking list pop)
3. Parses vote JSON
4. Inserts or updates vote in the database
5. Infinite loop until service stops

### 3. Result Service (Node.js/Express)

**Port** : 5001

**Responsibilities** :
- Results display API
- Real-time updates via WebSocket (Socket.io)
- PostgreSQL connection for querying votes

**Main files** :
- `result/server.js` : Express server
- `result/views/index.html` : Results HTML interface
- `result/views/app.js` : Client-side logic
- `result/views/socket.io.js` : Socket.io script

**npm Dependencies** :
- `express` : Web framework
- `pg` : PostgreSQL driver
- `socket.io` : Real-time communication
- `async` : Async utilities
- `body-parser`, `cookie-parser`, `method-override`

**Environment variables** :
- `POSTGRES_HOST`, `POSTGRES_PORT`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`

**Flow** :
1. Connects to PostgreSQL with automatic retry
2. Initializes WebSockets
3. Regularly fetches: `SELECT vote, COUNT(id) AS count FROM votes GROUP BY vote`
4. Sends results to connected clients via Socket.io
5. Clients display results in real-time

## 🗄️ Database Schema

### Table: `votes`

```sql
CREATE TABLE IF NOT EXISTS votes (
  id text PRIMARY KEY,        -- unique voter_id
  vote text NOT NULL          -- voting option
);
```

**Example data** :
```
id                | vote
------------------+-------
a1b2c3d4e5f6g7h8 | Ansible
x9y8z7w6v5u4t3s2 | Chef
```

## 🔧 Advanced Configuration

### Custom environment variables

Modify voting options in the Poll service:
```env
OPTION_A=Option1
OPTION_B=Option2
OPTION_C=Option3
OPTION_D=Option4
```

### Custom ports

Modify `compose.yml`:
```yaml
poll:
  ports:
    - "8000:80"  # New port

result:
  ports:
    - "8001:80"  # New port
```

### Data persistence

The `db-data` volume persists PostgreSQL data between restarts.

## 📊 Complete Data Flow

1. **Vote submission**
   - Browser → Poll Service (HTTP POST)
   - Poll Service → Redis Queue (vote JSON)

2. **Vote processing**
   - Worker → Redis Queue (blpop)
   - Worker → PostgreSQL (INSERT/UPDATE)

3. **Results display**
   - Browser → Result Service (HTTP GET + WebSocket)
   - Result Service → PostgreSQL (SELECT query)
   - Result Service → Browser (Socket.io update)

## 🌐 Docker Networks

The application uses 3 isolated networks:

- **poll-tier** : Poll ↔ Redis
- **back-tier** : Worker ↔ Redis, Worker ↔ PostgreSQL
- **result-tier** : Result ↔ PostgreSQL

This segmentation improves security and scalability.

## 📝 Logs and Debugging

### View logs from all services
```bash
docker-compose logs -f
```

### Logs from a specific service
```bash
docker-compose logs -f poll
docker-compose logs -f worker
docker-compose logs -f result
```

### Access Redis console
```bash
docker-compose exec redis redis-cli
# View vote queue
LRANGE votes 0 -1
# See number of votes
LLEN votes
```

### Access PostgreSQL console
```bash
docker-compose exec db psql -U postgres -d votes
# View votes
SELECT * FROM votes;
# Count votes by option
SELECT vote, COUNT(*) FROM votes GROUP BY vote;
```

## 🛑 Stopping and Cleanup

```bash
# Stop containers
docker-compose down

# Stop and remove volumes
docker-compose down -v

# Rebuild images
docker-compose build --no-cache
```

## 🐛 Troubleshooting

### Votes are not being processed
- Check that the Worker service is active: `docker-compose ps`
- Check Worker logs: `docker-compose logs worker`
- Check Redis connection: `docker-compose logs redis`

### Results are not displaying
- Verify that PostgreSQL is accessible
- Check Result logs: `docker-compose logs result`
- Check WebSocket connection in browser console

### Connection errors
- Services will attempt to reconnect automatically
- Verify all services are `running`: `docker-compose ps`

## 📚 Technology Stack

| Component | Technology | Version |
|-----------|------------|---------|
| **Poll Frontend** | Python | 3.x |
| **Poll Framework** | Flask | - |
| **Result Frontend** | Node.js | - |
| **Result Framework** | Express | 4.18.2 |
| **Real-time** | Socket.io | 4.7.4 |
| **Worker** | Java | 8+ |
| **Build Manager** | Maven | 3.x |
| **Cache/Queue** | Redis | 7 (Alpine) |
| **Database** | PostgreSQL | 15 (Alpine) |
| **Orchestration** | Docker Compose | - |

## 👤 Author

Joseph Marie Bile - T-DOP-501-NAN

## 📄 License

MIT

## 🤝 Contributing

Contributions are welcome. For major changes, please first open an issue to discuss the proposed modifications.