## DynamoDB Single Table Design -- Access Patterns & Schema


## Entity Types

### 1. User Profile

#### Access Patterns

1. **Get user profile by Discord ID**  
   `PK: USER#{discordId}` `SK: PROFILE`

2. **Update user's WFH counter for the month**  
   `PK: USER#{discordId}` `SK: PROFILE`

3. **Batch get multiple user profiles (for team member lists)**  
   BatchGetItem with multiple `PK: USER#{discordId}` `SK: PROFILE`

#### DB Schema

**PK:** `USER#{discordId}`  
**SK:** `PROFILE`

**Attributes:**
- `discordId` (String) — Discord snowflake ID
- `name` (String) — User's full name
- `email` (String) — User's email address
- `role` (String) — ADMIN | LEAD | LOGISTICS | EMPLOYEE
- `status` (String) — ACTIVE | INACTIVE
- `teamId` (String) — Foreign key to team
- `teamName` (String) — Denormalized team name
- `wfhCount` (Number) — Count of WFH days in current month
- `wfhMonth` (String) — YYYY-MM format, resets monthly
- `createdAt` (String) — ISO 8601 timestamp
- `updatedAt` (String) — ISO 8601 timestamp

**Schema Conventions:**
- `USER#` prefix identifies user partition
- `PROFILE` is a constant SK for user metadata
- Discord ID is the primary identifier (direct access via PK, no GSI needed)

---

### 2. Meal Record

#### Access Patterns

1. **Get user's single meal record for a specific date**  
   `PK: USER#{discordId}` `SK: RECORD#{YYYY-MM-DD}`

2. **Get user's 7-day meal records (today + 6 days)**  
   Query `PK: USER#{discordId}` `SK BETWEEN RECORD#{today} AND RECORD#{+6days}`

3. **Create or update meal record for a user**  
   `PK: USER#{discordId}` `SK: RECORD#{YYYY-MM-DD}`

4. **Get all meal records for a specific date (headcount aggregation summary)**  
   Query GSI1 `gsi1pk: DATE#{YYYY-MM-DD}`

5. **Batch create meal records (nightly cron job)**  
   `BatchWriteItem` with multiple `PK: USER#{discordId}` `SK: RECORD#{YYYY-MM-DD}`

#### DB Schema

**PK:** `USER#{discordId}`  
**SK:** `RECORD#{YYYY-MM-DD}`  
**gsi1pk:** `DATE#{YYYY-MM-DD}`  
**gsi1sk:** `USER#{discordId}`

**Attributes:**
- `lunch` (Boolean | null) — Opted into lunch
- `snacks` (Boolean | null) — Opted into snacks
- `iftar` (Boolean | null) — Opted into iftar
- `eventDinner` (Boolean | null) — Opted into event dinner
- `optionalDinner` (Boolean | null) — Opted into optional dinner
- `workFromHome` (Boolean) — Working from home flag
- `teamId` (String) — Denormalized for headcount grouping
- `teamName` (String) — Denormalized for headcount grouping
- `lastModifiedBy` (String) — discordId or 'SYSTEM'
- `createdAt` (String) — ISO 8601 timestamp
- `updatedAt` (String) — ISO 8601 timestamp

**Schema Conventions:**
- `RECORD#` prefix + date enables range queries for 7-day views
- `DATE#` prefix in GSI1 enables querying all users' records for a specific date
- Meal fields are nullable to distinguish "not set" from "opted out"
- `workFromHome` is always boolean (never null)

---

### 3. Active Users List

#### Access Patterns

1. **Get all active user Discord IDs (used by nightly cron)**  
   `PK: SYSTEM` `SK: ACTIVE_USERS`

2. **Update active users list when users are added/removed**  
   UpdateItem `PK: SYSTEM` `SK: ACTIVE_USERS` (ADD/DELETE from StringSet)

#### DB Schema

**PK:** `SYSTEM`  
**SK:** `ACTIVE_USERS`

**Attributes:**
- `memberIds` (StringSet) — Set of all active Discord IDs
- `updatedAt` (String) — ISO 8601 timestamp

**Schema Conventions:**
- `SYSTEM` partition holds system-wide metadata
- `ACTIVE_USERS` is a sentinel item to avoid full table scans
- StringSet enables efficient ADD/DELETE operations
- Cron job reads this once to get all active users

---

### 4. Meal Schedule

#### Access Patterns

1. **Get schedule for a specific date**  
   `PK: SCHEDULE#{YYYY-MM-DD}` `SK: METADATA`

2. **Create schedule for a date**  
   `PK: SCHEDULE#{YYYY-MM-DD}` `SK: METADATA`

3. **Update schedule for a date**  
   UpdateItem `PK: SCHEDULE#{YYYY-MM-DD}` `SK: METADATA`

4. **Delete schedule for a date**  
   DeleteItem `PK: SCHEDULE#{YYYY-MM-DD}` `SK: METADATA`

5. **List all upcoming schedules (scan with filter)**  
   Scan with filter `PK begins_with SCHEDULE# AND date >= today`

#### DB Schema

**PK:** `SCHEDULE#{YYYY-MM-DD}`  
**SK:** `METADATA`

**Attributes:**
- `date` (String) — YYYY-MM-DD
- `lunchEnabled` (Boolean) — Lunch available
- `snacksEnabled` (Boolean) — Snacks available
- `iftarEnabled` (Boolean) — Iftar available
- `eventDinnerEnabled` (Boolean) — Event dinner available
- `optionalDinnerEnabled` (Boolean) — Optional dinner available
- `occasionName` (String | null) — Optional occasion label
- `createdAt` (String) — ISO 8601 timestamp
- `updatedAt` (String) — ISO 8601 timestamp

**Schema Conventions:**
- `SCHEDULE#` prefix + date as PK enables direct GetItem by date
- `METADATA` is a constant SK for schedule configuration
- Date in PK allows scan filtering for upcoming schedules
- Low volume (~30 items max) makes scan acceptable


---
