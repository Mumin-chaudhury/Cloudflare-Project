# Product Feedback Aggregator

A serverless product feedback aggregation and analysis tool built on Cloudflare's Developer Platform.

## Overview

This tool aggregates mock product feedback from multiple sources (support tickets, Discord, GitHub, email, Twitter, forums), analyzes it using AI for sentiment and themes, and presents actionable insights to Product Managers through a clean dashboard.

## Architecture

### Cloudflare Products Used

1. **Workers** - Core serverless runtime hosting the API and dashboard
2. **D1** - SQLite database storing normalized feedback and analysis results
3. **Queues** - Asynchronous processing of feedback analysis
4. **Cron Triggers** - Scheduled daily digest generation
5. **KV** - Caching analysis results and storing daily digests
6. **AI (Workers AI)** - Sentiment analysis and theme extraction

### Data Flow

```
Mock Sources → Worker API (Ingest) → Queue → Consumer Worker (Analysis) → D1 + KV
                                                         ↓
                                              Cron Trigger (Daily Digest)
                                                         ↓
                                              Dashboard Worker (UI)
```

1. **Ingestion**: API endpoint receives feedback from mock sources, normalizes data, writes to D1, and queues for analysis
2. **Analysis**: Queue consumer performs AI-powered sentiment analysis and theme extraction, stores results
3. **Digest**: Daily cron trigger aggregates insights and stores summary in KV
4. **Dashboard**: Web UI fetches data from D1/KV and displays insights

### Why This Stack?

- **D1**: Perfect for structured feedback data with relational queries (filter by source, date, sentiment)
- **Queues**: Decouples ingestion from expensive AI analysis, prevents timeouts
- **KV**: Fast caching layer for frequently accessed dashboards and daily digests
- **Cron**: Automated daily summaries without manual intervention
- **Workers AI**: Built-in sentiment analysis without external API dependencies

## Project Structure

```
feedback-aggregator/
├── src/
│   ├── index.ts              # Main dashboard worker
│   ├── api.ts                # Ingestion API worker
│   ├── consumer.ts           # Queue consumer for analysis
│   ├── cron.ts               # Daily digest generator
│   ├── types.ts              # TypeScript types
│   └── templates/
│       └── dashboard.html    # Dashboard UI template
├── migrations/
│   └── 0001_initial.sql      # D1 schema setup
├── scripts/
│   └── seed-data.ts          # Mock data generator
├── wrangler.toml             # Cloudflare configuration
├── package.json
├── tsconfig.json
└── README.md
```

## Setup Instructions

### Prerequisites

- Node.js 18+ and npm
- Cloudflare account
- Wrangler CLI installed globally: `npm install -g wrangler`

### Local Development

1. **Clone and install dependencies**
   ```bash
   npm install
   ```

2. **Authenticate with Cloudflare**
   ```bash
   wrangler login
   ```

3. **Create D1 database**
   ```bash
   wrangler d1 create feedback-db
   ```
   
   Copy the database ID from output and update `wrangler.toml`:
   ```toml
   database_id = "your-database-id-here"
   ```

4. **Create Queue**
   ```bash
   wrangler queues create feedback-analysis-queue
   ```

5. **Run database migrations**
   ```bash
   wrangler d1 execute feedback-db --file=./migrations/0001_initial.sql
   ```

6. **Seed mock data (optional)**
   ```bash
   npm run seed
   ```

7. **Run locally**
   ```bash
   npm run dev
   ```
   
   The dashboard will be available at `http://localhost:8787`

## Deployment

### Deploy to Cloudflare

1. **Deploy all workers**
   ```bash
   npm run deploy
   ```

2. **Run migrations on production database**
   ```bash
   wrangler d1 execute feedback-db --remote --file=./migrations/0001_initial.sql
   ```

3. **Seed production data (optional)**
   ```bash
   npm run seed:production
   ```

### GitHub Integration

1. **Create repository**
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   git branch -M main
   git remote add origin https://github.com/yourusername/feedback-aggregator.git
   git push -u origin main
   ```

2. **Set up GitHub Actions** (optional)
   
   Add Cloudflare API token to GitHub Secrets as `CLOUDFLARE_API_TOKEN`
   
   The included `.github/workflows/deploy.yml` will auto-deploy on push to main.

## Usage

### Ingest Feedback

**POST** `/api/feedback`

```json
{
  "source": "discord",
  "content": "The new feature is amazing but the UI is confusing",
  "author": "user123",
  "metadata": {
    "channel": "feedback",
    "url": "https://discord.gg/..."
  }
}
```

### View Dashboard

Navigate to your worker URL (e.g., `https://feedback-aggregator.yoursubdomain.workers.dev`)

The dashboard shows:
- Recent feedback items with sentiment scores
- Theme clustering and top themes
- Source breakdown
- Sentiment distribution
- Urgency indicators
- Daily digest summaries

### API Endpoints

- `GET /` - Dashboard UI
- `GET /api/feedback` - List all feedback (supports filters)
- `POST /api/feedback` - Ingest new feedback
- `GET /api/digest` - Get latest daily digest
- `GET /api/themes` - Get top themes
- `GET /api/stats` - Get summary statistics

## Key Features

✅ Multi-source feedback aggregation  
✅ AI-powered sentiment analysis  
✅ Automatic theme extraction  
✅ Urgency scoring based on keywords  
✅ Daily automated digests  
✅ Clean, PM-friendly dashboard  
✅ Real-time insights  
✅ Fully serverless architecture  

## Development Commands

```bash
npm run dev          # Local development
npm run deploy       # Deploy to Cloudflare
npm run seed         # Seed local database
npm run tail         # View production logs
npm run types        # Generate TypeScript types
```

## Future Enhancements

- Real integrations with Discord, GitHub, Zendesk APIs
- More sophisticated ML clustering algorithms
- Slack/Discord notification webhooks
- Export to CSV/Excel
- User authentication and multi-tenant support
- Historical trend analysis
- Custom tagging and categorization

## License

MIT
