# Replay Hunter 🎯

**Hunt Down CS2 Demos** - Automatic demo file downloads via Steam Game Coordinator

A standalone REST API service for automatically downloading CS2 demo files using match sharecodes.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Node.js](https://img.shields.io/badge/Node.js-18+-green.svg)](https://nodejs.org/)

## Features

- 🎮 **Steam/CS2 Integration** - Direct connection to CS2 Game Coordinator
- 📥 **Automatic Downloads** - Demos are automatically downloaded and stored
- 🗄️ **PostgreSQL Database** - Robust storage of demo metadata
- 📊 **REST API** - Simple HTTP interface for managing demos
- 🔄 **Download Queue** - Bull + Redis for reliable background jobs
- 🧹 **Auto-Cleanup** - Automatic deletion of old demos
- 🔔 **Webhooks** - Notifications when demos are ready

## Quick Start

### Prerequisites

- Node.js 18+
- PostgreSQL
- Redis
- Steam Account with CS2

### Installation

```bash
# Clone repository
git clone https://github.com/meinjens/replay-hunter.git
cd replay-hunter

# Install dependencies
npm install

# Create database
createdb csdemos

# Configure environment
cp .env.example .env
# Edit .env with your credentials

# Run migrations
npx prisma migrate dev --name init
npx prisma generate

# Start Redis
redis-server

# Start service
npm start
```

The service will run on \`http://localhost:3000\`

## Environment Variables

```env
PORT=3000
DATABASE_URL=postgresql://user:password@localhost:5432/csdemos
REDIS_URL=redis://localhost:6379
STEAM_USERNAME=your_steam_username
STEAM_PASSWORD=your_steam_password
DEMOS_PATH=./demos
CLEANUP_ENABLED=true
CLEANUP_DAYS=30
```

## API Endpoints

### POST /api/demos

Request a demo download

**Request:**
```json
{
  "sharecode": "CSGO-xxxxx-xxxxx-xxxxx-xxxxx-xxxxx"
}
```

**Response:**
```json
{
  "id": "uuid",
  "sharecode": "CSGO-xxxxx-xxxxx-xxxxx-xxxxx-xxxxx",
  "status": "PENDING",
  "createdAt": "2025-01-27T12:00:00.000Z"
}
```

### GET /api/demos

List all demos

**Query Parameters:**
- \`status\`: Filter by status (PENDING, DOWNLOADING, COMPLETED, FAILED)
- \`limit\`: Number of results (default: 50)
- \`offset\`: Offset for pagination (default: 0)

### GET /api/demos/stats

Get statistics

**Response:**
```json
{
  "total": 100,
  "pending": 5,
  "downloading": 2,
  "completed": 90,
  "failed": 3
}
```

### GET /api/demos/:id

Get demo details

### GET /api/demos/:id/file

Download demo file (.dem.bz2)

### DELETE /api/demos/:id

Delete demo (including file)

## Project Structure

```
replay-hunter/
├── package.json
├── .env.example
├── .gitignore
├── README.md
├── prisma/
│   └── schema.prisma          # Database schema
├── src/
│   ├── index.js               # Server entry point
│   ├── config.js              # Configuration
│   ├── steam/
│   │   └── gcClient.js        # Steam/GC wrapper
│   ├── queue/
│   │   ├── downloadQueue.js   # Bull queue setup
│   │   └── worker.js          # Download worker
│   ├── services/
│   │   ├── demoService.js     # Demo business logic
│   │   ├── cleanupService.js  # Auto-cleanup
│   │   └── webhookService.js  # Webhook notifications
│   ├── routes/
│   │   └── demos.js           # API routes
│   └── utils/
│       ├── logger.js          # Logging
│       └── errors.js          # Error handling
└── demos/                     # Demo file storage
```

## Workflow

1. **User**: \`POST /api/demos\` with sharecode
2. **Service**:
   - Creates demo entry in DB (status: \`PENDING\`)
   - Adds to download queue
   - Returns immediate response
3. **Worker**:
   - Connects to Steam/GC
   - Fetches demo URL (status: \`FETCHING_URL\`)
   - Downloads .dem.bz2 file (status: \`DOWNLOADING\`)
   - Saves to \`./demos/\` (status: \`COMPLETED\`)
4. **User**: \`GET /api/demos/:id\` - Check status
5. **User**: \`GET /api/demos/:id/file\` - Download file

## Webhooks

When enabled, the service sends POST requests to the configured webhook URL when a demo is ready.

**Configuration in \`.env\`:**
```env
WEBHOOK_ENABLED=true
WEBHOOK_URL=http://example.com/webhook
WEBHOOK_SECRET=your_secret
```

**Webhook Payload:**
```json
{
  "event": "demo.completed",
  "demoId": "uuid",
  "sharecode": "...",
  "matchId": "...",
  "status": "COMPLETED",
  "downloadedAt": "2025-01-27T12:05:00.000Z"
}
```

The \`X-Webhook-Signature\` header contains an HMAC-SHA256 hash of the payload with the configured secret.

## Auto-Cleanup

The service automatically deletes old demos after a configured number of days.

**Configuration in \`.env\`:**
```env
CLEANUP_ENABLED=true
CLEANUP_DAYS=30
CLEANUP_INTERVAL_HOURS=24
```

## Development

### Development Mode with Auto-Reload

```bash
npm run dev
```

### Prisma Studio (Database GUI)

```bash
npm run prisma:studio
```

### Logs

Logs are stored in \`logs/\`:
- \`error.log\`: Errors only
- \`combined.log\`: All logs

## Troubleshooting

### "Failed to connect to Steam/GC"

- Check Steam credentials in \`.env\`
- Ensure the account owns CS2
- The CS2 GC might be offline (rare)

### "No demo URL found"

- Match is older than 30 days (demos are deleted)
- Match is not a matchmaking match
- Sharecode is invalid

### Redis Connection Error

```bash
# macOS
brew services start redis

# Linux
sudo systemctl start redis
```

### PostgreSQL Connection Error

```bash
# macOS
brew services start postgresql

# Linux
sudo systemctl start postgresql
```

## Integration

This service was originally part of [CStatSentry](https://github.com/meinjens/cstatsentry), a CS2 Anti-Cheat Detection System.

You can integrate Replay Hunter with your own projects:

```javascript
// Example: Request a demo download
const response = await fetch('http://localhost:3000/api/demos', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    sharecode: 'CSGO-xxxxx-xxxxx-xxxxx-xxxxx-xxxxx'
  })
});

const demo = await response.json();
console.log("Demo ID: ${demo.id}");

// Poll for completion
const checkStatus = async (id) => {
  const res = await fetch(\`http://localhost:3000/api/demos/\${id}\`);
  const data = await res.json();
  return data.status === 'COMPLETED';
};
```

## Contributing

1. Fork the repository
2. Create a feature branch: \`git checkout -b feature/amazing-feature\`
3. Commit your changes: \`git commit -m 'Add amazing feature'\`
4. Push to the branch: \`git push origin feature/amazing-feature\`
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Support

- 🐛 [Report Issues](https://github.com/meinjens/replay-hunter/issues)
- 💬 [Discussions](https://github.com/meinjens/replay-hunter/discussions)

---

**Made with ❤️ for the CS2 community**
