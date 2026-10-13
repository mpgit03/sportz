README.md
# Sportz — Real-Time Sports Broadcasting Backend

A real-time sports broadcasting backend built with **Node.js, Express 5, PostgreSQL, Drizzle ORM, and WebSockets**. Sportz provides REST APIs for match and commentary management while delivering live commentary updates to subscribed clients through persistent WebSocket connections.

The project explores real-time communication, database-backed APIs, event broadcasting, input validation, and reliable WebSocket connection management.

## Features

- **Match Management:** Create matches and retrieve match listings through REST APIs.
- **Commentary Management:** Store and retrieve match-specific play-by-play commentary.
- **Real-Time Broadcasting:** Broadcast commentary updates to clients subscribed to a match.
- **WebSocket Subscriptions:** Subscribe, unsubscribe, and manage multiple match subscriptions.
- **Connection Heartbeats:** Handle ping/pong messages for connection health checks.
- **Input Validation:** Validate REST requests and WebSocket messages using Zod.
- **Rate Limiting:** Restrict excessive WebSocket message traffic.
- **Backpressure Protection:** Protect against excessive buffering when clients consume updates slowly.
- **Subscription Limits:** Restrict the number of match subscriptions per connection.
- **Database Migrations:** Manage database schema changes with Drizzle ORM and Drizzle Kit.
- **Development Utilities:** Seed match data and simulate commentary updates for testing.

## Tech Stack

| Technology | Purpose |
|---|---|
| Node.js | JavaScript runtime |
| Express 5 | HTTP server and REST APIs |
| WebSockets (`ws`) | Real-time, bidirectional communication |
| PostgreSQL | Relational database |
| Drizzle ORM | Database queries and schema management |
| Drizzle Kit | Database migrations |
| Zod | Runtime input validation |
| Arcjet | Application security integration |
| dotenv | Environment variable management |

## Architecture

```text
                 ┌─────────────────────┐
                 │    Client Apps      │
                 └──────────┬──────────┘
                            │
                ┌───────────┴───────────┐
                │                       │
           REST / HTTP              WebSocket
                │                       │
                ▼                       ▼
        ┌──────────────────────────────────┐
        │          Express Server          │
        │                                  │
        │  Match APIs                      │
        │  Commentary APIs                 │
        │  Input Validation                │
        │  Subscription Management         │
        │  Real-Time Broadcasting          │
        │  Connection Safeguards           │
        └─────────────────┬────────────────┘
                          │
                          ▼
                  ┌───────────────┐
                  │  Drizzle ORM  │
                  └───────┬───────┘
                          │
                          ▼
                  ┌───────────────┐
                  │  PostgreSQL  │
                  └───────────────┘
```

### How it works

1. Clients retrieve match information through REST endpoints.
2. Clients establish a persistent WebSocket connection.
3. A client subscribes to one or more matches.
4. Commentary updates are broadcast to the relevant subscribers.
5. Clients can modify their subscriptions or send heartbeat messages.
6. Validation and connection safeguards help maintain reliable communication.

## Getting Started

### Prerequisites

- [Node.js](https://nodejs.org/)
- npm
- [Git](https://git-scm.com/)
- A running PostgreSQL database

### 1. Clone the repository

```bash
git clone https://github.com/mpgit03/sportz.git
cd sportz
```

### 2. Install dependencies

```bash
npm install
```

### 3. Configure environment variables

Create a `.env` file in the project root:

```env
# Database
DATABASE_URL="your_postgresql_connection_string"

# Server
PORT=8000
HOST=0.0.0.0

# Arcjet
ARCJET_KEY="your_arcjet_key"
ARCJET_ENV="development"

# API URL
API_URL="http://localhost:8000"

# Broadcasting configuration
BROADCAST=1
DELAY_MS=250
MATCH_COUNT=0
```

Replace the example values with your actual configuration. These variable names and default values should be checked against the application code before running the project.

**Security:** Never commit `.env` or expose database credentials and API keys in your repository.

### 4. Set up the database

Configure `DATABASE_URL` to point to your PostgreSQL database.

If your `package.json` and `drizzle.config.js` use the standard Drizzle Kit migration workflow, apply migrations with:

```bash
npx drizzle-kit migrate
```

Ensure your database is running and the migration files are available before executing this command.

### 5. Start the server

Run the development script defined in `package.json`:

```bash
npm run dev
```

Check the server logs and application configuration for the actual HTTP and WebSocket addresses.

## REST API

The following endpoints describe the project's API interface.

### List matches

```http
GET /matches?limit=50
```

Retrieves a list of matches, optionally limited by the `limit` query parameter.

### Create a match

```http
POST /matches
Content-Type: application/json
```

Example request:

```json
{
  "sport": "football",
  "homeTeam": "FC Neon",
  "awayTeam": "Drizzle United",
  "startTime": "2027-02-01T12:00:00.000Z",
  "endTime": "2027-02-01T13:45:00.000Z"
}
```

Supported status values include `scheduled`, `live`, and `finished`. Match status is computed from its start and end times.

### Retrieve match commentary

```http
GET /matches/:id/commentary?limit=100
```

Retrieves commentary entries associated with a particular match.

### Create commentary

```http
POST /matches/:id/commentary
Content-Type: application/json
```

Example request:

```json
{
  "minute": 42,
  "sequence": 120,
  "period": "2nd half",
  "eventType": "goal",
  "actor": "Alex Morgan",
  "team": "FC Neon",
  "message": "GOAL! Powerful finish from the edge of the box.",
  "metadata": {
    "assist": "Sam Kerr"
  },
  "tags": ["goal", "shot"]
}
```

The teams, players, and events in these examples are illustrative.

## WebSocket Protocol

### Connection

Connect to the WebSocket endpoint:

```text
ws://localhost:8000/ws
```

The port must match your server configuration.

To subscribe to a match when connecting, the protocol also supports a match ID query parameter:

```text
ws://localhost:8000/ws?matchId=123
```

### Client-to-server messages

**Subscribe to a match**

```json
{
  "type": "subscribe",
  "matchId": 123
}
```

**Unsubscribe from a match**

```json
{
  "type": "unsubscribe",
  "matchId": 123
}
```

**Update multiple subscriptions**

```json
{
  "type": "setSubscriptions",
  "matchIds": [1, 2, 3]
}
```

**Send a heartbeat ping**

```json
{
  "type": "ping"
}
```

### Server-to-client messages

**Connection established**

```json
{
  "type": "welcome"
}
```

**Subscription confirmed**

```json
{
  "type": "subscribed",
  "matchId": 123
}
```

**Unsubscription confirmed**

```json
{
  "type": "unsubscribed",
  "matchId": 123
}
```

**Commentary broadcast**

```json
{
  "type": "commentary",
  "data": {
    "id": 1,
    "matchId": 123,
    "message": "A new match event has occurred."
  }
}
```

**Heartbeat response**

```json
{
  "type": "pong"
}
```

The server can also return subscription lists and structured error messages.

### WebSocket safeguards

The documented implementation includes these connection limits:

| Safeguard | Limit |
|---|---:|
| Maximum subscriptions per connection | 50 |
| Message rate | 10 messages/second |
| Burst allowance | 20 messages |
| Maximum message payload | 1 MB |
| Backpressure threshold | 1 MB of buffered data |

These values describe the documented configuration and can be adjusted in the implementation.

## Project Structure

```text
sportz/
├── drizzle/                # Database migrations
├── public/                 # Static assets
├── src/                    # Application source code
├── drizzle.config.js       # Drizzle Kit configuration
├── package.json            # Scripts and dependencies
├── package-lock.json       # Dependency lockfile
├── test-ws-arcjet.js        # WebSocket/security test utility
├── .gitignore              # Git exclusions
└── README.md               # Project documentation
```

The exact module organization is maintained within `src/`.

## Engineering Considerations

### Real-time communication

Persistent WebSocket connections allow the server to send updates without requiring clients to repeatedly poll REST endpoints.

### Validation and reliability

Zod schemas help validate incoming messages and API payloads. Rate limits, subscription caps, heartbeats, and backpressure checks help protect the server from malformed traffic and resource exhaustion.

### Horizontal scaling

A WebSocket server can directly broadcast to clients connected to that instance. A multi-instance deployment would need a shared event-distribution mechanism, such as Redis Pub/Sub or NATS, to deliver events across instances.

### Security

Authentication and authorization are not included in the documented WebSocket protocol. A production deployment should authenticate clients, enforce access policies, and protect secrets through environment-based configuration.

## Future Improvements

- Add authentication and authorization for clients.
- Introduce shared event distribution for multi-instance deployments.
- Add automated integration tests and concurrent-client load tests.
- Implement structured logging, metrics, and health checks.
- Add graceful shutdown and production deployment configuration.
- Expand API documentation with client integration examples.

## Key Learning Outcomes

- Building REST APIs with Express.
- Implementing WebSocket connections and subscription management.
- Broadcasting real-time events to connected clients.
- Integrating PostgreSQL and database migrations.
- Applying runtime input validation.
- Handling rate limiting and WebSocket backpressure.
- Understanding the challenges of scaling stateful connections.

## Acknowledgements

This project was developed as a hands-on learning project inspired by the [JavaScript Mastery WebSockets tutorial](https://www.youtube.com/@javascriptmastery).

The repository documents my implementation and exploration of real-time backend engineering concepts.

---

**Built with Node.js, PostgreSQL, and WebSockets.**