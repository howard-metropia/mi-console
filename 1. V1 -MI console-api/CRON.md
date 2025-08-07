# Admin Platform API Cron Jobs Documentation

**Last Updated**: 2025-08-07  
**Version**: 1.0.0  
**System**: Admin Platform API - Cron Job Scheduler

---

## Table of Contents
1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Core Jobs Documentation](#core-jobs-documentation)
   - [update_campaign_status](#1-update_campaign_status)
   - [make_incentive_campaign](#2-make_incentive_campaign)
   - [send_give_away](#3-send_give_away)
4. [Execution & Monitoring](#execution--monitoring)
5. [Database Schema Reference](#database-schema-reference)
6. [Troubleshooting Guide](#troubleshooting-guide)

---

## Overview

The Admin Platform API uses a sophisticated cron-based job scheduling system to automate campaign lifecycle management, reward distribution, and system integration tasks. Jobs are executed at regular intervals using Node.js cron expressions.

### Key Features
- **Automated Campaign Lifecycle**: Campaigns automatically transition between states
- **External System Integration**: Seamless sync with V2 Incentive API
- **Reward Distribution**: Automated token and coin distribution
- **High Reliability**: Transaction safety and error recovery
- **Flexible Scheduling**: Configurable cron expressions

### Job Locations
- **Implementation**: `/src/jobs/` directory
- **Schedule Configuration**: `/src/schedule.js`
- **Database Models**: `/src/models/`
- **Utilities**: `/src/core/`

---

## System Architecture

### Execution Flow
```
app.js schedule → schedule.js → individual job files → database operations
                                                     → external API calls
                                                     → email notifications
```

### Database Connections
- **admin**: Primary admin platform database
- **portal**: User and trip data
- **incentive**: Incentive system integration

### External Integrations
- **Incentive Admin API**: `https://dev-incentive-admin-api.connectsmartx.com`
- **Token Distribution API**: External token management system
- **Email Service**: Status change notifications

---

## Core Jobs Documentation

## 1. update_campaign_status

### Overview
Monitors and automatically updates campaign statuses based on their scheduled start and end dates. This ensures campaigns flow through their lifecycle stages without manual intervention.

### Schedule
- **Cron Expression**: `*/10 * * * *`
- **Frequency**: Every 10 minutes
- **Execution Time**: ~5-30 seconds depending on campaign count

### Purpose
- Activate campaigns when their start date arrives
- Complete campaigns when their end date passes
- Enable coin tracking for active campaigns
- Send status change notifications

### Data Sources

#### Input Tables
```sql
-- Primary source: campaigns table
SELECT * FROM campaigns 
WHERE (status = 1 AND start_date <= NOW()) 
   OR (status = 2 AND end_date < NOW())
   OR (status = 4 AND end_date < NOW());

-- Related data
campaign_segments (audience targeting)
message_templates (notification content)
give_aways (reward configurations)
```

### Process Flow

#### Phase 1: Upcoming → In Progress
```javascript
// 1. Find campaigns ready to start
campaigns = await Campaign.query()
  .where('status', Campaign.status.upcoming)
  .where('start_date', '<=', currentUTC)
  .eager('[giveAway, challenge, raffle]');

// 2. Update each campaign
for (campaign of campaigns) {
  // Update status
  await campaign.$query().patch({
    status: Campaign.status.inProgress,
    count_coin_usage: true
  });
  
  // Special handling for Give Away campaigns
  if (campaign.type === 1 && campaign.giveAway) {
    if (giveAwayEndDate > campaignEndDate) {
      await campaign.$query().patch({
        end_date: giveAwayEndDate
      });
    }
  }
  
  // Update coin quantities
  await updateCoinByCampaignId(campaign.id);
  
  // Send notification
  await sendStatusChangeEmail(campaign, 'upcoming', 'in_progress');
}
```

#### Phase 2: In Progress → Completed
```javascript
// 1. Find campaigns ready to complete
campaigns = await Campaign.query()
  .where('status', Campaign.status.inProgress)
  .where('end_date', '<', currentUTC);

// 2. Update status
for (campaign of campaigns) {
  await campaign.$query().patch({
    status: Campaign.status.completed
  });
  
  await updateCoinByCampaignId(campaign.id);
  await sendStatusChangeEmail(campaign, 'in_progress', 'completed');
}
```

### Outputs

#### Database Changes
- **campaigns.status**: Updated to new lifecycle stage
- **campaigns.count_coin_usage**: Enabled for active campaigns
- **campaigns.end_date**: Extended for give away campaigns if needed
- **Coin inventory**: Updated via `updateCoinByCampaignId()`

#### Email Notifications
```json
{
  "subject": "Campaign Status Change Notification",
  "body": {
    "campaign_id": 1234,
    "campaign_name": "Summer Challenge",
    "old_status": "Upcoming",
    "new_status": "In Progress",
    "start_date": "2025-08-07 00:00:00",
    "end_date": "2025-08-21 23:59:59",
    "system_user": "cron_job"
  }
}
```

### Error Handling
- Database transactions ensure atomic updates
- Failed email notifications don't block status updates
- Logs capture update counts for monitoring
- Connection cleanup prevents resource leaks

### Status Code Reference
```javascript
Campaign.status = {
  draft: 0,      // Not processed by this job
  upcoming: 1,   // Waiting for start_date
  inProgress: 2, // Active campaign
  completed: 3,  // Past end_date
  forceStop: 4   // Manually stopped
}
```

---

## 2. make_incentive_campaign

### Overview
Creates and synchronizes bingocard-based incentive campaigns with the external V2 Incentive API. This job bridges the admin platform campaigns with the incentive system, enabling gamified user engagement.

### Schedule
- **Cron Expression**: `*/10 * * * *`
- **Frequency**: Every 10 minutes
- **Execution Time**: ~10-60 seconds depending on user count

### Purpose
- Create incentive campaigns from admin platform challenge campaigns
- Generate user bingo cards for campaign participants
- Track participation counts and synchronization status
- Support multiple bingocard formats and configurations

### Data Sources

#### Input Sources
```sql
-- Admin Database
SELECT * FROM campaigns 
WHERE type = 3 AND status = 2; -- Challenge campaigns in progress

SELECT * FROM campaign_segments;
SELECT * FROM bingocard_types;
SELECT * FROM incentive_campaign_mapping;

-- Portal Database  
SELECT user_id, email FROM auth_user;
SELECT * FROM segment_endpoints;

-- Incentive API
GET /api/v2/campaigns
GET /api/v2/bingocard-types
POST /api/v2/campaigns
POST /api/v2/user-bingocards
```

### Process Flow

#### Phase 1: Campaign Creation
```javascript
// 1. Get running incentive campaigns
const runningCampaigns = await axios.get(
  `${incentiveAdminUrl}/api/v2/campaigns?is_expired=false`
);

// 2. Find unmapped admin campaigns
const unmappedCampaigns = await Campaign.query()
  .where('type', 3)
  .where('status', 2)
  .whereNotIn('id', mappedCampaignIds);

// 3. Create incentive campaign for each
for (campaign of unmappedCampaigns) {
  // Extract challenge configuration
  const genWeight = parseGenWeight(campaign.challenge);
  
  // Create via API
  const payload = {
    campaign: {
      name: campaign.name,
      reward_type: mapRewardType(challenge.prize_type),
      reward_amount: challenge.quantity_per_gift,
      start_time: campaign.start_date,
      end_time: campaign.end_date
    },
    bingocard: {
      gen_weight: genWeight,
      width: challenge.width || 3,
      height: challenge.height || 3,
      required_line: challenge.required_line || 1,
      game_duration: Math.min(challenge.duration || 7, 365)
    }
  };
  
  const response = await axios.post(
    `${incentiveAdminUrl}/api/v2/campaigns`,
    payload
  );
  
  // Store mapping
  await IncentiveCampaignMapping.create({
    campaign_id: campaign.id,
    incentive_campaign_id: response.data.id
  });
}
```

#### Phase 2: User Bingo Card Assignment
```javascript
// 1. Get bingocard type with smart detection
const bingocardTypeId = await getBingocardTypeId(
  incentiveCampaign,
  genWeight,
  challenge
);

// 2. Get campaign users
const segmentUsers = await getSegmentUsers(campaign.segmentIds);

// 3. Filter users needing cards
const usersNeedingCards = segmentUsers.filter(user => 
  !existingUserIds.includes(user.user_id) &&
  !isInternalUser(user.email)
);

// 4. Create bingo cards
const userBingocardPayload = {
  user_ids: usersNeedingCards.map(u => u.user_id),
  bingocard_type_id: bingocardTypeId,
  permissions: {
    calendar: true,
    notification: true
  }
};

await axios.post(
  `${incentiveAdminUrl}/api/v2/campaigns/${incentiveCampaignId}/user-bingocards`,
  userBingocardPayload
);
```

#### Phase 3: Smart Bingocard Type Detection
```javascript
// Sophisticated parsing logic handles multiple formats:
function parseGenWeight(challenge) {
  // 1. Structured format: "id:3x3:1:7:20"
  if (genWeight && genWeight.includes(':')) {
    const parts = genWeight.split(':');
    return {
      id: parts[0],
      grid: parts[1],
      requiredLine: parts[2],
      duration: parts[3],
      objective: parts[4]
    };
  }
  
  // 2. Legacy string: "make_trips", "transit", etc.
  const legacyMap = {
    'make_trips': 1,
    'transit': 2,
    'bike_walk': 3,
    'carpool': 4,
    'green': 5,
    'referral': 6
  };
  
  // 3. Custom format: "custom_20"
  if (genWeight?.startsWith('custom_')) {
    return genWeight.split('_')[1];
  }
  
  // 4. Direct objective ID
  return challenge.objective || genWeight;
}
```

### Outputs

#### External API Changes
- **New Incentive Campaigns**: Created with bingocard configurations
- **User Bingo Cards**: Assigned to segment users with permissions
- **Campaign Status**: Tracked in incentive system

#### Database Updates
```sql
-- Mapping records
INSERT INTO incentive_campaign_mapping 
(campaign_id, incentive_campaign_id, created_at)
VALUES (?, ?, NOW());

-- User count tracking
UPDATE campaigns 
SET incentive_user_count = ? 
WHERE id = ?;
```

### Error Handling
- **API Failures**: Logged with full request/response details
- **Invalid Configurations**: Skipped with warning logs
- **Duration Limits**: Capped at 365 days maximum
- **Missing Bingocard Types**: Fallback to parameter matching
- **User Filtering**: Excludes test/internal accounts

### Configuration Mappings
```javascript
// Prize Type → Reward Type
const rewardTypeMap = {
  0: 'token',      // External tokens
  1: 'merchandise', // Physical prizes  
  2: 'coin',       // Platform coins
  3: 'logo'        // Branded items
};

// Legacy Objective Names
const objectiveMap = {
  'make_trips': 'Any trip type',
  'transit': 'Public transit only',
  'bike_walk': 'Active transportation',
  'carpool': 'Shared rides',
  'green': 'Eco-friendly modes',
  'referral': 'User referrals'
};
```

---

## 3. send_give_away

### Overview
Automatically distributes rewards (tokens or coins) to users who complete qualifying actions during give-away campaigns. This job monitors user activities and processes reward distribution based on campaign rules.

### Schedule
- **Cron Expression**: `*/10 * * * *`
- **Frequency**: Every 10 minutes
- **Execution Time**: ~5-45 seconds depending on user/trip count
- **Processing Window**: Last 600 seconds (10 minutes)

### Purpose
- Monitor user trips and activities
- Identify users meeting campaign criteria
- Distribute tokens via external API
- Credit coins directly to user accounts
- Track winners and prevent duplicates

### Data Sources

#### Input Sources
```sql
-- Admin Database
SELECT * FROM campaigns 
WHERE type = 1 AND status = 2 
AND give_away.prize_type IN (0, 2); -- Tokens or Coins

SELECT * FROM give_aways;
SELECT * FROM campaign_winners;
SELECT * FROM tokens;
SELECT * FROM coins;

-- Portal Database
SELECT * FROM trip 
WHERE created_at >= NOW() - INTERVAL 10 MINUTE;

SELECT * FROM telework 
WHERE date >= CURRENT_DATE - INTERVAL 7 DAY;

SELECT * FROM auth_user;
SELECT * FROM enterprise_account_member;
```

### Process Flow

#### Phase 1: Campaign Selection
```javascript
// 1. Get active give-away campaigns
const campaigns = await Campaign.query()
  .where('type', 1)  // Give Away
  .where('status', 2) // In Progress
  .whereIn('giveAway.prize_type', [0, 2]) // Token or Coin
  .eager('[giveAway, campaignWinners, segments]');

// 2. Filter by date range
const eligibleCampaigns = campaigns.filter(campaign => {
  const now = new Date();
  return (campaign.start_date <= now && campaign.end_date >= now) ||
         campaign.giveAway.all_time === 1;
});

// 3. Separate by repeatability
const repeatableCampaigns = eligibleCampaigns.filter(c => 
  c.giveAway.can_repeat === 1
);
const nonRepeatableCampaigns = eligibleCampaigns.filter(c => 
  c.giveAway.can_repeat !== 1
);
```

#### Phase 2: User Qualification
```javascript
// 1. Get eligible users
let users = await getSegmentUsers(campaign.segments);

// Filter by organization if specified
if (campaign.giveAway.org_id) {
  users = users.filter(u => 
    u.organization_id === campaign.giveAway.org_id
  );
}

// 2. Check action completion
const qualifyingUsers = [];
for (user of users) {
  // Count trips/telework by action_id
  const actionCount = await countUserActions(
    user.id,
    campaign.giveAway.action_id,
    campaign.giveAway.start_date,
    campaign.giveAway.end_date
  );
  
  if (actionCount >= campaign.giveAway.minimum_count) {
    // Calculate reward multiplier for repeatable campaigns
    const rewardMultiplier = campaign.giveAway.can_repeat ? 
      Math.floor(actionCount / campaign.giveAway.minimum_count) : 1;
    
    qualifyingUsers.push({
      user_id: user.id,
      email: user.email,
      reward_count: rewardMultiplier
    });
  }
}

// 3. Exclude previous winners (non-repeatable)
if (!campaign.giveAway.can_repeat) {
  const previousWinners = await CampaignWinner.query()
    .where('campaign_id', campaign.id);
  
  qualifyingUsers = qualifyingUsers.filter(u =>
    !previousWinners.find(w => w.user_id === u.user_id)
  );
}
```

#### Phase 3: Token Distribution
```javascript
// 1. Verify inventory
const availableTokens = token.quantity - token.distributed;
const maxRecipients = Math.floor(
  availableTokens / campaign.giveAway.quantity_per_gift
);

// 2. Prepare recipients
const recipients = qualifyingUsers.slice(0, maxRecipients);

// 3. Call Token API
const tokenPayload = {
  agency_id: campaign.giveAway.token_or_group_id,
  agency_name: campaign.giveAway.token_or_group_name,
  token_id: token.id,
  token_name: token.name,
  campaign_id: campaign.id,
  campaign_name: campaign.name,
  token_initial_date: campaign.giveAway.initiate_date,
  token_expire_date: campaign.giveAway.expire_date,
  zone: campaign.giveAway.timezone || 'UTC',
  token_per_winner: campaign.giveAway.quantity_per_gift,
  user_ids: recipients.map(r => r.user_id)
};

const response = await sendTokensToUsers(tokenPayload);

// 4. Update inventory
if (response.success) {
  await token.$query().patch({
    distributed: token.distributed + 
      (recipients.length * campaign.giveAway.quantity_per_gift)
  });
}
```

#### Phase 4: Coin Distribution
```javascript
// 1. Check coin balance
const availableCoins = await getCoinBalance(
  campaign.giveAway.coin_source_id
);

// 2. Process each user
for (user of qualifyingUsers) {
  const coinAmount = campaign.giveAway.quantity_per_gift * 
                     user.reward_count;
  
  if (availableCoins >= coinAmount) {
    // Create transaction
    const transaction = await createPointsTransaction({
      user_id: user.user_id,
      points: coinAmount,
      type: 'give_away',
      campaign_id: campaign.id,
      description: `Give Away: ${campaign.name}`
    });
    
    // Record winner
    await CampaignWinner.create({
      user_id: user.user_id,
      campaign_id: campaign.id,
      type: 2, // Coin
      quantity: coinAmount,
      prize_date: new Date(),
      transaction_id: transaction.id
    });
    
    // Send notification
    await sendPushNotification(user.user_id, {
      title: 'Coins Received!',
      body: `Auto refill completed! ${coinAmount} Coins have been deposited in your Wallet.`
    });
    
    availableCoins -= coinAmount;
  }
}
```

#### Phase 5: Repeat Tracking
```javascript
// For repeatable campaigns, track used trips
if (campaign.giveAway.can_repeat) {
  for (winner of processedWinners) {
    // Get trips used for this reward
    const usedTrips = await getTripsForReward(
      winner.user_id,
      campaign.giveAway.action_id,
      campaign.giveAway.minimum_count
    );
    
    // Mark trips as used
    await markTripsAsUsed(
      usedTrips,
      campaign.id,
      winner.transaction_id
    );
  }
}
```

### Outputs

#### External Systems
- **Token API**: Batch token distribution requests
- **Push Notifications**: Coin receipt confirmations
- **Email Service**: Optional winner notifications

#### Database Updates
```sql
-- Winner records
INSERT INTO campaign_winners 
(user_id, campaign_id, type, quantity, prize_date, transaction_id)
VALUES (?, ?, ?, ?, NOW(), ?);

-- Token inventory
UPDATE tokens 
SET distributed = distributed + ? 
WHERE id = ?;

-- Points transactions (for coins)
INSERT INTO points_transactions
(user_id, points, type, reference_id, created_at)
VALUES (?, ?, 'give_away', ?, NOW());

-- Repeat tracking
INSERT INTO give_away_repeats
(user_id, campaign_id, trip_id, processed_at)
VALUES (?, ?, ?, NOW());
```

### Error Handling
- **Token API Failures**: Individual user retry with backoff
- **Insufficient Inventory**: Partial distribution with logging
- **Transaction Failures**: Rollback with error tracking
- **Duplicate Prevention**: Database constraints and pre-checks
- **User Validation**: Email and eligibility verification

### Action Type Reference
```javascript
// Travel mode mapping
const actionTypes = {
  1: 'carpool',
  2: 'bike',
  3: 'transit',
  4: 'telework',
  5: 'car',
  6: 'walk',
  8: 'telework',
  
  // Carpool specific
  100: 'carpool_driver',
  101: 'carpool_rider',
  102: 'driver_carpool',
  103: 'rider_carpool',
  104: 'driver_only',
  105: 'rider_only',
  106: 'any_carpool'
};

// Query logic by action
function getActionQuery(actionId, userId, startDate, endDate) {
  switch(actionId) {
    case 1: // Carpool
      return `mode_id = 1`;
    case 2: // Bike
      return `mode_id = 2`;
    case 3: // Transit
      return `mode_id IN (3, 10, 11)`; // Bus, Rail, Subway
    case 100: // Driver
      return `mode_id = 1 AND is_driver = true`;
    // ... etc
  }
}
```

---

## Execution & Monitoring

### Starting the Scheduler
```bash
# Production
npm run schedule

# Development with auto-restart
npm run schedule:dev

# Single job test
./DEV_0729/SERVER/test-cron-job.sh <job_name>
```

### Monitoring Commands
```bash
# Real-time log monitoring
tail -f logs/cron-jobs.log | grep -E "(update_campaign_status|make_incentive_campaign|send_give_away)"

# Check job execution history
./DEV_0729/SERVER/check-job-history.sh

# Monitor campaign status changes
./DEV_0729/SERVER/watch-campaign-status.sh
```

### Log Locations
- **General Logs**: `/logs/cron-jobs.log`
- **Error Logs**: `/logs/cron-errors.log`
- **API Logs**: `/logs/external-api.log`

### Health Checks
```javascript
// Job health endpoint
GET /api/v1/admin/cron-health

// Response
{
  "status": "healthy",
  "jobs": {
    "update_campaign_status": {
      "last_run": "2025-08-07T10:20:00Z",
      "status": "success",
      "campaigns_updated": 5
    },
    "make_incentive_campaign": {
      "last_run": "2025-08-07T10:20:00Z",
      "status": "success",
      "campaigns_created": 2,
      "users_assigned": 150
    },
    "send_give_away": {
      "last_run": "2025-08-07T10:20:00Z",
      "status": "success",
      "tokens_sent": 50,
      "coins_distributed": 500
    }
  }
}
```

---

## Database Schema Reference

### Core Tables

#### campaigns
```sql
CREATE TABLE campaigns (
  id INT PRIMARY KEY,
  name VARCHAR(255),
  type INT, -- 1: Give Away, 2: Raffle, 3: Challenge
  status INT, -- 0: Draft, 1: Upcoming, 2: In Progress, 3: Completed, 4: Force Stop
  start_date DATETIME,
  end_date DATETIME,
  count_coin_usage BOOLEAN DEFAULT FALSE,
  created_at DATETIME,
  updated_at DATETIME
);
```

#### give_aways
```sql
CREATE TABLE give_aways (
  id INT PRIMARY KEY,
  campaign_id INT,
  prize_type INT, -- 0: Token, 1: Merchandise, 2: Coin, 3: Logo
  action_id INT, -- Travel mode or action type
  minimum_count INT DEFAULT 1,
  quantity_per_gift DECIMAL(10,2),
  can_repeat BOOLEAN DEFAULT FALSE,
  org_id INT, -- Optional organization filter
  start_date DATETIME,
  end_date DATETIME,
  all_time BOOLEAN DEFAULT FALSE
);
```

#### campaign_winners
```sql
CREATE TABLE campaign_winners (
  id INT PRIMARY KEY,
  user_id INT,
  campaign_id INT,
  type INT, -- Prize type
  quantity DECIMAL(10,2),
  prize_date DATETIME,
  transaction_id VARCHAR(100),
  created_at DATETIME
);
```

#### incentive_campaign_mapping
```sql
CREATE TABLE incentive_campaign_mapping (
  id INT PRIMARY KEY,
  campaign_id INT,
  incentive_campaign_id INT,
  created_at DATETIME,
  UNIQUE KEY (campaign_id, incentive_campaign_id)
);
```

---

## Troubleshooting Guide

### Common Issues and Solutions

#### 1. Campaigns Not Transitioning Status
**Symptoms**: Campaigns remain in "Upcoming" past start date

**Checks**:
- Verify cron job is running: `ps aux | grep schedule`
- Check timezone settings in database vs server
- Review logs for errors: `grep update_campaign_status logs/cron-jobs.log`

**Solution**:
```bash
# Manual status update
npm run job:update-campaign-status
```

#### 2. Incentive Campaigns Not Created
**Symptoms**: Challenge campaigns have no incentive mapping

**Checks**:
- Verify API endpoint connectivity
- Check gen_weight format in challenge data
- Review bingocard type availability

**Debug**:
```javascript
// Test API connection
curl -X GET https://dev-incentive-admin-api.connectsmartx.com/api/v2/health

// Check challenge data
SELECT id, name, challenge FROM campaigns WHERE type = 3 AND status = 2;
```

#### 3. Rewards Not Distributed
**Symptoms**: Qualifying users not receiving tokens/coins

**Checks**:
- Verify inventory levels
- Check user eligibility queries
- Review winner history for duplicates

**Manual Distribution**:
```bash
# Force give-away processing
npm run job:send-give-away -- --campaign-id=123
```

### Performance Optimization

#### Database Indexes
```sql
-- Critical indexes for job performance
CREATE INDEX idx_campaign_status_dates ON campaigns(status, start_date, end_date);
CREATE INDEX idx_trip_user_date ON trip(user_id, created_at);
CREATE INDEX idx_winners_campaign_user ON campaign_winners(campaign_id, user_id);
```

#### Job Timing
- Stagger job execution to prevent overlap
- Increase intervals for low-activity periods
- Use database connection pooling

### Emergency Procedures

#### Stop All Jobs
```bash
# Kill scheduler process
pkill -f "node.*schedule"

# Disable in configuration
echo "ENABLE_CRON=false" >> .env
```

#### Rollback Campaign Status
```sql
-- Revert campaign to previous status
UPDATE campaigns 
SET status = 1, updated_at = NOW() 
WHERE id = ? AND status = 2;
```

#### Clear Job Locks
```sql
-- Remove stale job locks
DELETE FROM job_locks 
WHERE locked_at < NOW() - INTERVAL 1 HOUR;
```

---

## Appendix: Complete Cron Schedule

```javascript
// From src/schedule.js
const schedule = {
  // High frequency (every 10 minutes)
  'update_campaign_status': '*/10 * * * *',
  'make_incentive_campaign': '*/10 * * * *',
  'send_give_away': '*/10 * * * *',
  
  // Hourly jobs
  'update_incentive_success_record': '35 * * * *',
  'segment_update': '50 * * * *',
  'check_segment_delete_auth_user': '15 * * * *',
  
  // Daily jobs
  'dashboard_summary': '30 7 * * *',
  'campaign_once': '1 14 * * *',
  'campaign_daily': '2 14 * * *',
  'campaign_weekly': '3 14 * * *',
  'campaign_monthly': '4 14 * * *',
  
  // Maintenance jobs
  'delete_old_update_record': '10 8 1 */4 *',
  'delete_old_aws_segment_endpoint_log': '20 8 1 */2 *'
};
```

---

**Document Version**: 1.0.0  
**Last Updated**: 2025-08-07  
**Maintained By**: Admin Platform Development Team