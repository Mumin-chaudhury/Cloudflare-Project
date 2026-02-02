# Testing Guide

Comprehensive testing guide for the Product Feedback Aggregator.

## Test Environment Setup

### Local Testing

```bash
# Start local development server
npm run dev

# In another terminal, run tests
npm test
```

### Testing Checklist

Before deploying to production, verify:

- [ ] Database schema created successfully
- [ ] Sample data loaded
- [ ] API endpoints responding
- [ ] Dashboard loads without errors
- [ ] AI analysis processes feedback
- [ ] Queue consumer working
- [ ] Cron trigger configured (check with `wrangler deployments list`)

## Unit Testing

### Testing Database Operations

```bash
# Check feedback count
wrangler d1 execute feedback-db --command="SELECT COUNT(*) as count FROM feedback"

# Verify sentiment distribution
wrangler d1 execute feedback-db --command="SELECT sentiment_label, COUNT(*) FROM feedback GROUP BY sentiment_label"

# Check theme frequency
wrangler d1 execute feedback-db --command="SELECT * FROM theme_frequency ORDER BY count DESC LIMIT 5"

# View recent feedback
wrangler d1 execute feedback-db --command="SELECT * FROM feedback ORDER BY created_at DESC LIMIT 3"
```

### Testing Queue Processing

```bash
# Submit test feedback and watch queue processing
curl -X POST http://localhost:8787/api/feedback \
  -H "Content-Type: application/json" \
  -d '{
    "source": "discord",
    "content": "This is a test message with URGENT priority!",
    "author": "test_user"
  }'

# Check analysis log to verify processing
wrangler d1 execute feedback-db --command="SELECT * FROM analysis_log ORDER BY analyzed_at DESC LIMIT 3"
```

## Integration Testing

### Test API Endpoints

#### 1. Test Feedback Ingestion

```bash
# Positive feedback
curl -X POST http://localhost:8787/api/feedback \
  -H "Content-Type: application/json" \
  -d '{
    "source": "support_ticket",
    "content": "The new dashboard is amazing! Love the redesign.",
    "author": "happy_user@example.com",
    "metadata": {"ticket_id": "TEST-001"}
  }'

# Negative feedback
curl -X POST http://localhost:8787/api/feedback \
  -H "Content-Type: application/json" \
  -d '{
    "source": "github",
    "content": "CRITICAL BUG: Application crashes on startup!",
    "author": "github_user_123",
    "metadata": {"issue_number": 999}
  }'

# Neutral feedback
curl -X POST http://localhost:8787/api/feedback \
  -H "Content-Type: application/json" \
  -d '{
    "source": "email",
    "content": "Question about pricing for enterprise plans",
    "author": "sales@company.com"
  }'
```

#### 2. Test Feedback Retrieval

```bash
# Get all feedback
curl http://localhost:8787/api/feedback

# Get with pagination
curl "http://localhost:8787/api/feedback?limit=5&offset=0"

# Filter by source
curl "http://localhost:8787/api/feedback?source=discord"

# Filter by sentiment
curl "http://localhost:8787/api/feedback?sentiment=negative"
```

#### 3. Test Statistics Endpoint

```bash
curl http://localhost:8787/api/stats | jq
```

Expected response structure:
```json
{
  "success": true,
  "data": {
    "total_feedback": 24,
    "last_24h": 3,
    "avg_sentiment": 0.15,
    "by_source": { "discord": 8, "github": 5, ... },
    "by_sentiment": { "positive": 12, "neutral": 8, "negative": 4 },
    "urgent_count": 2
  }
}
```

#### 4. Test Themes Endpoint

```bash
curl http://localhost:8787/api/themes | jq
```

#### 5. Test Digest Endpoint

```bash
curl http://localhost:8787/api/digest | jq
```

### Test Dashboard UI

1. Open http://localhost:8787 in browser
2. Verify sections load:
   - Statistics cards show numbers
   - Top themes display
   - Recent feedback visible
   - Daily digest section present
3. Check browser console for errors
4. Verify auto-refresh (wait 30 seconds)

## Load Testing

### Simple Load Test

```bash
# Install hey (HTTP load testing tool)
# macOS: brew install hey
# Linux: go install github.com/rakyll/hey@latest

# Test ingestion endpoint (100 requests, 10 concurrent)
hey -n 100 -c 10 -m POST \
  -H "Content-Type: application/json" \
  -d '{"source":"discord","content":"Load test","author":"test"}' \
  http://localhost:8787/api/feedback

# Test GET endpoint (1000 requests, 50 concurrent)
hey -n 1000 -c 50 http://localhost:8787/api/stats
```

Expected results:
- Ingestion: < 200ms p95 latency
- Stats endpoint: < 50ms p95 latency (cached)
- 0% error rate

### Stress Testing Queue

```bash
# Submit 100 feedback items rapidly
for i in {1..100}; do
  curl -X POST http://localhost:8787/api/feedback \
    -H "Content-Type: application/json" \
    -d "{\"source\":\"discord\",\"content\":\"Stress test $i\",\"author\":\"test_$i\"}" &
done
wait

# Monitor queue processing
wrangler queues list

# Check processed count
wrangler d1 execute feedback-db --command="SELECT COUNT(*) FROM feedback WHERE processed = 1"
```

## Testing AI Analysis

### Verify Sentiment Analysis

```bash
# Ingest feedback with clear sentiment
curl -X POST http://localhost:8787/api/feedback \
  -H "Content-Type: application/json" \
  -d '{
    "source": "twitter",
    "content": "This product is absolutely terrible and broken",
    "author": "angry_user"
  }'

# Wait 5 seconds for queue processing
sleep 5

# Check sentiment was detected
wrangler d1 execute feedback-db --command="SELECT content, sentiment_label, sentiment_score FROM feedback WHERE author = 'angry_user'"
```

Expected: `sentiment_label = 'negative'`, `sentiment_score < 0`

### Verify Theme Extraction

```bash
# Ingest feedback mentioning specific themes
curl -X POST http://localhost:8787/api/feedback \
  -H "Content-Type: application/json" \
  -d '{
    "source": "support_ticket",
    "content": "The UI is confusing and performance is slow",
    "author": "theme_test_user"
  }'

# Wait for processing
sleep 5

# Check themes
wrangler d1 execute feedback-db --command="SELECT themes FROM feedback WHERE author = 'theme_test_user'"
```

Expected: Themes array containing "UI/UX" and "Performance"

### Verify Urgency Scoring

```bash
# Ingest urgent feedback
curl -X POST http://localhost:8787/api/feedback \
  -H "Content-Type: application/json" \
  -d '{
    "source": "email",
    "content": "URGENT! CRITICAL BUG! System is completely broken and not working!",
    "author": "urgent_test"
  }'

# Check urgency score
sleep 5
wrangler d1 execute feedback-db --command="SELECT urgency_score FROM feedback WHERE author = 'urgent_test'"
```

Expected: `urgency_score >= 7`

## Testing Cron Trigger

### Manual Trigger Test

```bash
# Trigger the scheduled event manually (requires deployed worker)
wrangler triggers schedule feedback-aggregator "0 9 * * *" --test

# Check KV for digest
wrangler kv:key get "digest:$(date +%Y-%m-%d)" --binding=CACHE
```

### Verify Daily Digest Generation

```bash
# Check today's digest via API
curl http://localhost:8787/api/digest | jq

# Verify highlights are generated
curl http://localhost:8787/api/digest | jq '.data.highlights'
```

## Testing KV Cache

### Verify Cache Behavior

```bash
# First request (cache miss)
time curl http://localhost:8787/api/stats

# Second request (cache hit - should be faster)
time curl http://localhost:8787/api/stats

# Check KV directly
wrangler kv:key list --binding=CACHE
```

### Clear Cache for Testing

```bash
# Clear all KV keys
wrangler kv:key delete "stats:latest" --binding=CACHE
wrangler kv:key delete "digest:$(date +%Y-%m-%d)" --binding=CACHE
```

## Error Handling Tests

### Test Invalid Input

```bash
# Missing required fields
curl -X POST http://localhost:8787/api/feedback \
  -H "Content-Type: application/json" \
  -d '{"content":"Missing source"}'

# Expected: 400 Bad Request

# Invalid source type
curl -X POST http://localhost:8787/api/feedback \
  -H "Content-Type: application/json" \
  -d '{"source":"invalid_source","content":"Test","author":"user"}'

# Expected: 400 Bad Request
```

### Test Database Errors

```bash
# Try to access non-existent feedback
curl "http://localhost:8787/api/feedback?source=nonexistent"

# Should return empty array, not error
```

## Production Testing

### Pre-Deployment Checklist

```bash
# 1. Verify wrangler.toml has correct IDs
cat wrangler.toml | grep -E "(database_id|id =)"

# 2. Test build
npm run build

# 3. Dry-run deployment
wrangler deploy --dry-run

# 4. Check migrations are ready
ls -la migrations/

# 5. Verify secrets (if any)
wrangler secret list
```

### Post-Deployment Verification

```bash
# 1. Get deployed URL
wrangler deployments list

# 2. Test production API
curl https://your-worker.workers.dev/api/stats

# 3. Submit test feedback
curl -X POST https://your-worker.workers.dev/api/feedback \
  -H "Content-Type: application/json" \
  -d '{"source":"email","content":"Production test","author":"deploy_test"}'

# 4. Monitor logs
wrangler tail

# 5. Check metrics
# Visit Cloudflare Dashboard → Workers → feedback-aggregator → Metrics
```

## Automated Testing Script

Create `test.sh`:

```bash
#!/bin/bash

BASE_URL="${1:-http://localhost:8787}"

echo "Testing Feedback Aggregator at $BASE_URL"

# Test 1: Health check (stats endpoint)
echo "✓ Testing stats endpoint..."
curl -s "$BASE_URL/api/stats" | jq -e '.success == true' > /dev/null || exit 1

# Test 2: Submit feedback
echo "✓ Testing feedback ingestion..."
RESULT=$(curl -s -X POST "$BASE_URL/api/feedback" \
  -H "Content-Type: application/json" \
  -d '{"source":"discord","content":"Automated test","author":"test_bot"}')
echo $RESULT | jq -e '.success == true' > /dev/null || exit 1

# Test 3: Retrieve feedback
echo "✓ Testing feedback retrieval..."
curl -s "$BASE_URL/api/feedback?limit=5" | jq -e '.success == true' > /dev/null || exit 1

# Test 4: Get themes
echo "✓ Testing themes endpoint..."
curl -s "$BASE_URL/api/themes" | jq -e '.success == true' > /dev/null || exit 1

# Test 5: Get digest
echo "✓ Testing digest endpoint..."
curl -s "$BASE_URL/api/digest" | jq -e '.success == true' > /dev/null || exit 1

echo "✅ All tests passed!"
```

Run with:
```bash
chmod +x test.sh
./test.sh                                    # Test local
./test.sh https://your-worker.workers.dev    # Test production
```

## Monitoring & Debugging

### Real-time Log Monitoring

```bash
# Watch all logs
wrangler tail

# Filter for errors only
wrangler tail --format=json | jq 'select(.level == "error")'

# Filter for specific worker
wrangler tail --format=json | jq 'select(.scriptName == "feedback-aggregator")'
```

### Database Inspection

```bash
# Check database size
wrangler d1 info feedback-db

# Export all data for inspection
wrangler d1 execute feedback-db --command="SELECT * FROM feedback" --json > feedback_dump.json

# Check for stuck queue messages
wrangler d1 execute feedback-db --command="SELECT COUNT(*) FROM feedback WHERE processed = 0"
```

### Performance Monitoring

Key metrics to track:
- API response time (target: < 100ms p95)
- Queue processing time (target: < 2s per message)
- AI analysis accuracy (manual spot-check)
- Cache hit rate (target: > 80%)
- Error rate (target: < 0.1%)

## Troubleshooting Common Issues

### Issue: Feedback not being processed

**Check**:
```bash
wrangler d1 execute feedback-db --command="SELECT COUNT(*) FROM feedback WHERE processed = 0"
```

**Solution**: Manually trigger queue consumer or check queue depth

### Issue: Dashboard showing stale data

**Check**:
```bash
wrangler kv:key list --binding=CACHE
```

**Solution**: Clear cache or reduce TTL

### Issue: AI analysis failing

**Check**: Worker logs for AI errors
**Solution**: Verify Workers AI is enabled, check fallback logic

## Test Coverage Goals

- [x] API endpoints (100%)
- [x] Database operations (100%)
- [x] Queue processing (100%)
- [x] AI analysis (with fallback)
- [x] Cron triggers
- [x] Error handling
- [x] Cache behavior
- [ ] End-to-end user flows (manual testing)
- [ ] Browser compatibility (manual testing)
