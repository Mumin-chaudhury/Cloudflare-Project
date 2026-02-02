# API Documentation

Complete API reference for the Product Feedback Aggregator.

## Base URL

```
https://feedback-aggregator.yoursubdomain.workers.dev
```

## Authentication

Currently no authentication required. 

**Production Note**: Add API key authentication before production use.

## Endpoints

### 1. Ingest Feedback

Submit new feedback to the system.

**Endpoint**: `POST /api/feedback`

**Request Body**:
```json
{
  "source": "discord",
  "content": "The new feature is great but the UI is confusing",
  "author": "user123",
  "metadata": {
    "channel": "feedback",
    "url": "https://discord.gg/example"
  }
}
```

**Parameters**:
- `source` (required): One of `support_ticket`, `discord`, `github`, `email`, `twitter`, `forum`
- `content` (required): The feedback text (max 5000 chars recommended)
- `author` (required): Username, email, or identifier
- `metadata` (optional): Additional context as JSON object

**Response**: `201 Created`
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "created_at": "2026-02-01T14:30:00.000Z"
  }
}
```

**Example**:
```bash
curl -X POST https://your-worker.workers.dev/api/feedback \
  -H "Content-Type: application/json" \
  -d '{
    "source": "github",
    "content": "Feature request: Add dark mode support",
    "author": "github_user_42",
    "metadata": {
      "issue_number": 156,
      "labels": ["enhancement", "ui"]
    }
  }'
```

---

### 2. Get Feedback List

Retrieve paginated feedback with optional filters.

**Endpoint**: `GET /api/feedback`

**Query Parameters**:
- `limit` (optional): Items per page (default: 20, max: 100)
- `offset` (optional): Pagination offset (default: 0)
- `source` (optional): Filter by source type
- `sentiment` (optional): Filter by sentiment (`positive`, `neutral`, `negative`)

**Response**: `200 OK`
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440001",
        "source": "discord",
        "content": "Love the new dashboard redesign!",
        "author": "user123",
        "created_at": "2026-02-01T10:30:00.000Z",
        "metadata": "{\"channel\":\"feedback\"}",
        "sentiment_score": 0.85,
        "sentiment_label": "positive",
        "themes": "[\"UI/UX\",\"Features\"]",
        "urgency_score": 2,
        "processed": 1
      }
    ],
    "total": 42,
    "page": 1,
    "per_page": 20
  }
}
```

**Examples**:
```bash
# Get first 10 items
curl https://your-worker.workers.dev/api/feedback?limit=10

# Get negative feedback only
curl https://your-worker.workers.dev/api/feedback?sentiment=negative

# Get all Discord feedback
curl https://your-worker.workers.dev/api/feedback?source=discord

# Pagination (page 2)
curl https://your-worker.workers.dev/api/feedback?limit=20&offset=20
```

---

### 3. Get Statistics

Retrieve aggregate statistics and metrics.

**Endpoint**: `GET /api/stats`

**Response**: `200 OK`
```json
{
  "success": true,
  "data": {
    "total_feedback": 247,
    "last_24h": 18,
    "avg_sentiment": 0.34,
    "by_source": {
      "discord": 89,
      "github": 62,
      "support_ticket": 45,
      "email": 28,
      "twitter": 15,
      "forum": 8
    },
    "by_sentiment": {
      "positive": 124,
      "neutral": 78,
      "negative": 45
    },
    "urgent_count": 7
  }
}
```

**Example**:
```bash
curl https://your-worker.workers.dev/api/stats
```

---

### 4. Get Top Themes

Retrieve the most frequently mentioned themes.

**Endpoint**: `GET /api/themes`

**Response**: `200 OK`
```json
{
  "success": true,
  "data": [
    {
      "theme": "UI/UX",
      "count": 45,
      "avg_sentiment": 0.62
    },
    {
      "theme": "Performance",
      "count": 38,
      "avg_sentiment": -0.23
    },
    {
      "theme": "Features",
      "count": 32,
      "avg_sentiment": 0.51
    }
  ]
}
```

**Example**:
```bash
curl https://your-worker.workers.dev/api/themes
```

---

### 5. Get Daily Digest

Retrieve the latest daily digest summary.

**Endpoint**: `GET /api/digest`

**Response**: `200 OK`
```json
{
  "success": true,
  "data": {
    "date": "2026-02-01",
    "total_feedback": 18,
    "by_source": {
      "discord": 8,
      "github": 5,
      "support_ticket": 3,
      "email": 2
    },
    "by_sentiment": {
      "positive": 9,
      "neutral": 6,
      "negative": 3
    },
    "top_themes": [
      {
        "theme": "Features",
        "count": 7,
        "avg_sentiment": 0.45
      }
    ],
    "urgent_items": 2,
    "highlights": [
      "Received 18 pieces of feedback today",
      "2 items marked as urgent requiring immediate attention",
      "Top sentiment: positive",
      "Most active channel: discord"
    ]
  }
}
```

**Example**:
```bash
curl https://your-worker.workers.dev/api/digest
```

---

## Error Responses

### 400 Bad Request
```json
{
  "success": false,
  "error": "Invalid source type"
}
```

### 404 Not Found
```json
{
  "success": false,
  "error": "Not Found"
}
```

### 500 Internal Server Error
```json
{
  "success": false,
  "error": "Internal Server Error"
}
```

---

## Integration Examples

### Python Integration

```python
import requests

API_URL = "https://your-worker.workers.dev"

def submit_feedback(source, content, author, metadata=None):
    """Submit feedback to the aggregator"""
    response = requests.post(
        f"{API_URL}/api/feedback",
        json={
            "source": source,
            "content": content,
            "author": author,
            "metadata": metadata or {}
        }
    )
    return response.json()

# Example usage
result = submit_feedback(
    source="github",
    content="Bug: Login page not responsive on mobile",
    author="github_user_99",
    metadata={"issue_number": 234}
)
print(f"Created feedback: {result['data']['id']}")
```

### Node.js Integration

```javascript
const API_URL = "https://your-worker.workers.dev";

async function submitFeedback(source, content, author, metadata = {}) {
  const response = await fetch(`${API_URL}/api/feedback`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ source, content, author, metadata })
  });
  return response.json();
}

// Example usage
const result = await submitFeedback(
  'discord',
  'Feature request: Add export to CSV',
  'discord_user_456',
  { channel: 'feature-requests' }
);
console.log(`Created feedback: ${result.data.id}`);
```

### Discord Webhook Integration

```javascript
// Discord bot listening for messages
client.on('messageCreate', async (message) => {
  if (message.channel.name === 'feedback' && !message.author.bot) {
    await fetch('https://your-worker.workers.dev/api/feedback', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        source: 'discord',
        content: message.content,
        author: message.author.username,
        metadata: {
          channel: message.channel.name,
          server: message.guild.name,
          message_id: message.id
        }
      })
    });
    await message.react('✅');
  }
});
```

### GitHub Actions Integration

```yaml
# .github/workflows/feedback.yml
name: Submit GitHub Issues as Feedback

on:
  issues:
    types: [opened, edited]

jobs:
  submit:
    runs-on: ubuntu-latest
    steps:
      - name: Submit to Feedback Aggregator
        run: |
          curl -X POST https://your-worker.workers.dev/api/feedback \
            -H "Content-Type: application/json" \
            -d "{
              \"source\": \"github\",
              \"content\": \"${{ github.event.issue.title }}: ${{ github.event.issue.body }}\",
              \"author\": \"${{ github.event.issue.user.login }}\",
              \"metadata\": {
                \"issue_number\": ${{ github.event.issue.number }},
                \"labels\": ${{ toJSON(github.event.issue.labels) }}
              }
            }"
```

---

## Rate Limiting

Currently no rate limiting is enforced.

**Production Recommendation**: Implement rate limiting using Cloudflare's Rate Limiting rules:
- 100 requests per minute per IP for POST endpoints
- 1000 requests per minute per IP for GET endpoints

---

## Webhook Support (Future)

**Coming Soon**: Webhook notifications when:
- Urgent feedback is received (urgency_score > 8)
- Sentiment drops below threshold
- Specific keywords are detected

**Planned Endpoint**: `POST /api/webhooks`

---

## Best Practices

### 1. Batch Submissions
For high-volume sources, batch feedback submissions rather than real-time:

```javascript
const feedbackBatch = [];
// Collect feedback...
feedbackBatch.push({ source: 'discord', content: '...' });

// Submit every 5 minutes
setInterval(async () => {
  for (const item of feedbackBatch) {
    await submitFeedback(item);
  }
  feedbackBatch.length = 0;
}, 5 * 60 * 1000);
```

### 2. Error Handling
Always handle network errors and retries:

```javascript
async function submitWithRetry(data, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await submitFeedback(data);
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise(r => setTimeout(r, 1000 * Math.pow(2, i)));
    }
  }
}
```

### 3. Metadata Enrichment
Include as much context as possible in metadata:

```json
{
  "metadata": {
    "user_id": "12345",
    "plan": "enterprise",
    "url": "https://...",
    "browser": "Chrome",
    "os": "macOS",
    "feature": "dashboard"
  }
}
```

---

## Dashboard Access

The main dashboard is available at the root URL:

```
https://your-worker.workers.dev/
```

Features:
- Real-time statistics
- Recent feedback with sentiment indicators
- Top themes visualization
- Daily digest highlights
- Auto-refreshes every 30 seconds
