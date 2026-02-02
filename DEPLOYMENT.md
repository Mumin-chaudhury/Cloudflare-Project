# Deployment Checklist

Complete checklist for deploying the Product Feedback Aggregator to production.

## Pre-Deployment (Complete Before First Deploy)

### 1. Cloudflare Account Setup
- [ ] Cloudflare account created
- [ ] Wrangler CLI installed (`npm install -g wrangler`)
- [ ] Authenticated with Cloudflare (`wrangler login`)
- [ ] Account ID obtained (`wrangler whoami`)

### 2. Resource Creation
- [ ] D1 database created: `wrangler d1 create feedback-db`
- [ ] D1 database ID copied to `wrangler.toml`
- [ ] KV namespace created: `wrangler kv:namespace create CACHE`
- [ ] KV namespace ID copied to `wrangler.toml`
- [ ] Queue created: `wrangler queues create feedback-analysis-queue`
- [ ] Dead letter queue created: `wrangler queues create feedback-analysis-dlq`

### 3. Configuration
- [ ] `wrangler.toml` updated with all resource IDs
- [ ] Cron schedule verified (default: 9 AM UTC daily)
- [ ] Workers AI binding confirmed in config
- [ ] Environment-specific settings reviewed

### 4. Database Setup
- [ ] Schema migration file reviewed (`migrations/0001_initial.sql`)
- [ ] Migration tested locally
- [ ] Indexes verified for performance

### 5. Code Review
- [ ] All TypeScript files compile without errors
- [ ] No sensitive data hardcoded
- [ ] Error handling implemented
- [ ] CORS headers configured
- [ ] API rate limiting considered

### 6. Local Testing
- [ ] `npm run dev` works without errors
- [ ] Dashboard loads successfully
- [ ] API endpoints respond correctly
- [ ] Queue processes messages
- [ ] AI analysis completes
- [ ] Database queries execute

## First Deployment

### Step 1: Deploy Infrastructure
```bash
# Build the project
npm run build

# Deploy the worker
wrangler deploy
```

- [ ] Deployment successful
- [ ] Worker URL obtained
- [ ] No deployment errors

### Step 2: Initialize Database
```bash
# Run migrations on production database
wrangler d1 execute feedback-db --remote --file=./migrations/0001_initial.sql
```

- [ ] Migration executed successfully
- [ ] Tables created (verify with `wrangler d1 execute feedback-db --remote --command="SELECT name FROM sqlite_master WHERE type='table'"`)

### Step 3: Load Initial Data (Optional)
```bash
# Seed production with sample data
wrangler d1 execute feedback-db --remote --file=./scripts/seed.sql
```

- [ ] Sample data loaded
- [ ] Dashboard shows data

### Step 4: Verify Deployment
```bash
# Get worker URL
wrangler deployments list

# Test stats endpoint
curl https://your-worker.workers.dev/api/stats

# Test dashboard
open https://your-worker.workers.dev
```

- [ ] Stats endpoint returns valid JSON
- [ ] Dashboard loads without errors
- [ ] Sample feedback visible

### Step 5: Test Full Workflow
```bash
# Submit test feedback
curl -X POST https://your-worker.workers.dev/api/feedback \
  -H "Content-Type: application/json" \
  -d '{
    "source": "discord",
    "content": "Deployment test - this is urgent!",
    "author": "deploy_test_user"
  }'

# Wait 10 seconds for queue processing
sleep 10

# Verify it was processed
curl "https://your-worker.workers.dev/api/feedback?limit=1" | jq
```

- [ ] Feedback submitted successfully
- [ ] Queue processed the message
- [ ] Sentiment analysis completed
- [ ] Themes extracted
- [ ] Urgency score calculated

### Step 6: Configure Monitoring
```bash
# Start tailing logs (in separate terminal)
wrangler tail
```

- [ ] Logs are streaming
- [ ] No error messages appearing
- [ ] Request patterns look normal

## GitHub Integration (Optional)

### Step 1: Create Repository
```bash
git init
git add .
git commit -m "Initial commit: Product Feedback Aggregator"
git branch -M main
git remote add origin https://github.com/yourusername/feedback-aggregator.git
git push -u origin main
```

- [ ] Repository created
- [ ] Code pushed to GitHub
- [ ] README.md visible

### Step 2: Configure GitHub Actions
- [ ] Cloudflare API token created (`wrangler api-token create`)
- [ ] `CLOUDFLARE_API_TOKEN` added to GitHub Secrets
- [ ] `CLOUDFLARE_ACCOUNT_ID` added to GitHub Secrets
- [ ] `.github/workflows/deploy.yml` reviewed
- [ ] Test push triggers deployment

### Step 3: Verify Auto-Deployment
```bash
# Make a small change
echo "# Test" >> README.md
git add README.md
git commit -m "Test auto-deployment"
git push
```

- [ ] GitHub Action triggered
- [ ] Deployment successful
- [ ] Changes live in production

## Production Readiness

### Security
- [ ] HTTPS only (automatic with Workers)
- [ ] CORS configured appropriately
- [ ] Consider adding API authentication
- [ ] Review and limit exposed metadata
- [ ] Plan for API key rotation strategy

### Performance
- [ ] KV caching configured (5min TTL for stats)
- [ ] Database indexes created
- [ ] Queue batch size optimized (10 messages)
- [ ] Response times acceptable (< 200ms)

### Scalability
- [ ] Queue consumer auto-scales
- [ ] Database size limits understood (100GB max)
- [ ] KV storage limits known (unlimited)
- [ ] Workers request limits reviewed

### Reliability
- [ ] Dead letter queue configured
- [ ] Queue retry policy set (max 3 retries)
- [ ] Error handling in all endpoints
- [ ] Fallback sentiment analysis implemented

### Monitoring
- [ ] Cloudflare Analytics dashboard reviewed
- [ ] Key metrics identified
- [ ] Alert thresholds defined
- [ ] Log retention understood

### Documentation
- [ ] README.md complete
- [ ] ARCHITECTURE.md reviewed
- [ ] API.md accurate
- [ ] QUICKSTART.md tested
- [ ] Team onboarded

## Post-Deployment

### Within 1 Hour
- [ ] Monitor error rates
- [ ] Check queue depth
- [ ] Verify cron trigger scheduled
- [ ] Test all API endpoints
- [ ] Confirm dashboard accessible

### Within 24 Hours
- [ ] Review logs for errors
- [ ] Check daily digest generation
- [ ] Verify KV cache hit rate
- [ ] Test from multiple locations/devices
- [ ] Measure actual latencies

### Within 1 Week
- [ ] Analyze usage patterns
- [ ] Optimize query performance
- [ ] Review AI analysis accuracy
- [ ] Gather user feedback
- [ ] Plan next iteration

## Rollback Procedure

If issues are discovered:

### Quick Rollback
```bash
# Get previous deployment version
wrangler deployments list

# Rollback to previous version
wrangler rollback [VERSION_ID]
```

### Full Rollback
```bash
# Revert code changes
git revert HEAD

# Redeploy
wrangler deploy
```

### Database Rollback
```bash
# If schema change was problematic
wrangler d1 execute feedback-db --remote --file=./migrations/rollback.sql
```

## Environment-Specific Checklists

### Development
- [ ] Local database seeded
- [ ] Test data available
- [ ] Debug logging enabled

### Staging (Optional)
- [ ] Separate Cloudflare resources
- [ ] Copy of production data
- [ ] Limited access

### Production
- [ ] All checklists above completed
- [ ] Team notified of deployment
- [ ] Monitoring active
- [ ] Backup plan ready

## Maintenance Schedule

### Daily
- [ ] Check error logs
- [ ] Review digest generation
- [ ] Monitor queue depth

### Weekly
- [ ] Review analytics
- [ ] Check database growth
- [ ] Update documentation

### Monthly
- [ ] Security audit
- [ ] Performance review
- [ ] Cost analysis
- [ ] Feature planning

## Support Contacts

- Cloudflare Support: https://support.cloudflare.com
- Workers Documentation: https://developers.cloudflare.com/workers
- D1 Documentation: https://developers.cloudflare.com/d1
- Community Discord: https://discord.gg/cloudflaredev

## Success Criteria

Deployment is successful when:
- ✅ All API endpoints return 200/201
- ✅ Dashboard loads in < 2 seconds
- ✅ Zero error rate on core endpoints
- ✅ Queue processes messages within 30s
- ✅ AI analysis accuracy > 70%
- ✅ Cron job executes daily
- ✅ Cache hit rate > 80%
- ✅ No data loss or corruption

## Next Steps After Deployment

1. **Monitor Performance**: Watch Cloudflare Analytics for 48 hours
2. **Gather Feedback**: Share with team and collect feedback
3. **Add Integrations**: Connect real data sources (Discord, GitHub, etc.)
4. **Enhance Features**: Implement notification webhooks
5. **Scale Testing**: Run load tests to find limits
6. **Documentation**: Update docs based on production learnings

---

**Deployment Completed**: _______________
**Deployed By**: _______________
**Production URL**: _______________
**Notes**: _______________
