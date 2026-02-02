# Quick Start Guide

Get the Product Feedback Aggregator up and running in 10 minutes.

## Prerequisites Checklist

- [ ] Node.js 18+ installed
- [ ] Cloudflare account created
- [ ] Wrangler CLI installed: `npm install -g wrangler`
- [ ] Git installed (for GitHub integration)

## Step-by-Step Setup

### 1. Clone & Install (2 min)

```bash
# If from GitHub
git clone https://github.com/yourusername/feedback-aggregator.git
cd feedback-aggregator

# Install dependencies
npm install
```

### 2. Authenticate with Cloudflare (1 min)

```bash
wrangler login
```

This opens a browser window - click "Allow" to authorize Wrangler.

### 3. Create Cloudflare Resources (3 min)

```bash
# Create D1 Database
wrangler d1 create feedback-db

# Create KV Namespace
wrangler kv:namespace create CACHE

# Create Queue
wrangler queues create feedback-analysis-queue

# Create Dead Letter Queue
wrangler queues create feedback-analysis-dlq
```

**Important**: Copy the IDs returned by each command!

### 4. Update Configuration (2 min)

Edit `wrangler.toml` and replace the placeholder IDs:

```toml
[[d1_databases]]
binding = "DB"
database_name = "feedback-db"
database_id = "xxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx"  # ← Your D1 ID here

[[kv_namespaces]]
binding = "CACHE"
id = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"  # ← Your KV ID here
```

### 5. Initialize Database (1 min)

```bash
# Run migrations
wrangler d1 execute feedback-db --file=./migrations/0001_initial.sql

# Load sample data
wrangler d1 execute feedback-db --file=./scripts/seed.sql
```

### 6. Test Locally (1 min)

```bash
npm run dev
```

Open http://localhost:8787 in your browser. You should see the dashboard with sample data!

### 7. Deploy to Production (30 sec)

```bash
npm run deploy
```

Your app is now live! The URL will be shown in the output:
```
https://feedback-aggregator.yoursubdomain.workers.dev
```

## Verify Deployment

### Test the API

```bash
# Ingest a new feedback item
curl -X POST https://feedback-aggregator.yoursubdomain.workers.dev/api/feedback \
  -H "Content-Type: application/json" \
  -d '{
    "source": "discord",
    "content": "This is a test feedback from the API",
    "author": "test_user",
    "metadata": {"channel": "testing"}
  }'
```

### Check the Dashboard

Visit your worker URL in a browser. You should see:
- ✅ Statistics showing total feedback, sentiment averages
- ✅ Recent feedback items with sentiment labels
- ✅ Top themes extracted from feedback
- ✅ Daily digest section

## Common Issues

### "Database not found"

**Problem**: D1 database ID not updated in `wrangler.toml`

**Solution**: 
```bash
wrangler d1 list  # Find your database ID
# Update wrangler.toml with the correct ID
```

### "Queue binding error"

**Problem**: Queue not created or wrong name

**Solution**:
```bash
wrangler queues list  # Check queue exists
# Ensure queue name matches wrangler.toml
```

### "No data showing in dashboard"

**Problem**: Database not seeded

**Solution**:
```bash
wrangler d1 execute feedback-db --remote --file=./scripts/seed.sql
```

### Workers AI errors

**Problem**: AI binding not available

**Note**: Workers AI is automatically included. If errors persist, check:
```bash
wrangler whoami  # Verify account is in good standing
```

## Next Steps

### Add Real Data Sources

1. **GitHub Integration**: Use GitHub webhooks to send issues
2. **Discord Bot**: Create a bot to monitor channels
3. **Email Parser**: Forward emails to a Cloudflare Email Worker

### Customize Analysis

Edit `src/consumer.ts` to:
- Add custom theme categories
- Adjust urgency scoring weights
- Modify sentiment thresholds

### Enable Notifications

Add Slack webhook in `src/index.ts` scheduled function:

```typescript
async scheduled(event: ScheduledEvent, env: Env) {
  const digest = await generateDailyDigest(env);
  
  // Send to Slack
  await fetch('https://hooks.slack.com/services/YOUR/WEBHOOK/URL', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: `Daily Feedback Digest: ${digest.total_feedback} items`
    })
  });
}
```

## GitHub Actions (Optional)

### Setup Auto-Deploy on Push

1. Get your Cloudflare API token:
   ```bash
   wrangler api-token create
   ```

2. Add to GitHub Secrets:
   - Go to repo Settings → Secrets → Actions
   - Add `CLOUDFLARE_API_TOKEN`
   - Add `CLOUDFLARE_ACCOUNT_ID` (from `wrangler whoami`)

3. Push to main branch - auto-deployment will trigger!

## Monitoring

### View Real-Time Logs

```bash
wrangler tail
```

### Check D1 Database

```bash
wrangler d1 execute feedback-db --command="SELECT COUNT(*) FROM feedback"
```

### Monitor Queue Depth

```bash
wrangler queues list
```

## Need Help?

- 📖 [Cloudflare Workers Docs](https://developers.cloudflare.com/workers/)
- 📖 [D1 Documentation](https://developers.cloudflare.com/d1/)
- 📖 [Workers AI Guide](https://developers.cloudflare.com/workers-ai/)
- 🐛 [Report Issues](https://github.com/yourusername/feedback-aggregator/issues)

## Success Criteria

You know it's working when:
- ✅ Dashboard loads without errors
- ✅ Sample feedback is visible
- ✅ Sentiment scores are shown
- ✅ Themes are extracted and displayed
- ✅ API accepts new feedback (test with curl)
- ✅ Queue processes messages (check analysis_log table)

**Congratulations! Your feedback aggregator is running! 🎉**
