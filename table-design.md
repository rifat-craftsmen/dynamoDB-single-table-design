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

### 5. Team

#### Access Patterns

1. **Get team details and member list**  
   `PK: TEAM#{teamId}` `SK: METADATA`

2. **Get all team member profiles**  
   GetItem `PK: TEAM#{teamId}` `SK: METADATA` → BatchGetItem profiles

3. **Update team membership (add/remove members)**  
   UpdateItem `PK: TEAM#{teamId}` `SK: METADATA` (ADD/DELETE from memberIds StringSet)

#### DB Schema

**PK:** `TEAM#{teamId}`  
**SK:** `METADATA`

**Attributes:**
- `teamId` (String) — Unique team identifier
- `name` (String) — Team name
- `leadId` (String) — Discord ID of team lead
- `memberIds` (StringSet) — Set of active member Discord IDs
- `createdAt` (String) — ISO 8601 timestamp
- `updatedAt` (String) — ISO 8601 timestamp

**Schema Conventions:**
- `TEAM#` prefix identifies team partition
- `METADATA` is a constant SK for team configuration
- `memberIds` StringSet works for teams <100 members (400KB limit)
- Team lead's discordId stored for authorization checks

---

### 6. WFH Period

#### Access Patterns

1. **List all WFH periods**  
   Query `PK: WFHPERIOD`

2. **List all WFH periods sorted by start date (via GSI1)**  
   Query GSI1 `gsi1pk: WFHPERIOD` (sorted by gsi1sk)

3. **Create WFH period**  
   `PK: WFHPERIOD` `SK: {uuid}`

4. **Update WFH period**  
   UpdateItem `PK: WFHPERIOD` `SK: {uuid}`

5. **Delete WFH period**  
   DeleteItem `PK: WFHPERIOD` `SK: {uuid}`

6. **Check if a date falls within any WFH period**  
   Query `PK: WFHPERIOD` → filter in application (dateFrom <= date <= dateTo)

#### DB Schema

**PK:** `WFHPERIOD`  
**SK:** `{uuid}`  
**gsi1pk:** `WFHPERIOD`  
**gsi1sk:** `{dateFrom}#{uuid}`

**Attributes:**
- `id` (String) — UUID
- `dateFrom` (String) — YYYY-MM-DD start date
- `dateTo` (String) — YYYY-MM-DD end date
- `note` (String | null) — Optional description
- `createdAt` (String) — ISO 8601 timestamp
- `updatedAt` (String) — ISO 8601 timestamp

**Schema Conventions:**
- `WFHPERIOD` constant PK groups all periods together
- UUID as SK ensures uniqueness
- GSI1 with `dateFrom` in SK enables date-sorted queries
- Query PK='WFHPERIOD' returns all periods for date range filtering

---

### 7. Audit Log

#### Access Patterns

1. **Write audit entry on every mutation**  
   `PK: AUDIT#{entityType}#{entityId}` `SK: {timestamp}#{uuid}`

2. **Get all changes made by a specific user (actor)**  
   Query GSI1 `gsi1pk: AUDIT_BY_ACTOR#{actorDiscordId}`

3. **Get all changes made to a specific entity**  
   Query `PK: AUDIT#{entityType}#{entityId}`

4. **Get changes of a specific entity type by a user**  
   Query GSI2 `gsi2pk: AUDIT_BY_ACTOR_ENTITY#{actorDiscordId}#{entityType}`

#### DB Schema

**PK:** `AUDIT#{entityType}#{entityId}`  
**SK:** `{timestamp}#{uuid}`  
**gsi1pk:** `AUDIT_BY_ACTOR#{actorDiscordId}`  
**gsi1sk:** `{timestamp}#{uuid}`  
**gsi2pk:** `AUDIT_BY_ACTOR_ENTITY#{actorDiscordId}#{entityType}`  
**gsi2sk:** `{timestamp}#{uuid}`

**Attributes:**
- `id` (String) — UUID
- `timestamp` (String) — ISO 8601 timestamp
- `actorDiscordId` (String) — Who made the change
- `actorName` (String) — Denormalized actor name
- `action` (String) — CREATE | UPDATE | DELETE
- `entityType` (String) — USER | MEAL_RECORD | SCHEDULE | TEAM | WFH_PERIOD
- `entityId` (String) — Identifier of the changed entity
- `targetDiscordId` (String | null) — For USER or MEAL_RECORD changes
- `changes` (Object) — `{ field: { old: value, new: value } }`
- `metadata` (Object) — Optional context (IP, command name, etc.)

**Schema Conventions:**
- `AUDIT#` prefix + entity type + entity ID groups all changes to an entity
- Timestamp + UUID in SK provides chronological ordering and uniqueness
- `AUDIT_BY_ACTOR#` in GSI1 enables querying all actions by a user
- `AUDIT_BY_ACTOR_ENTITY#` in GSI2 enables filtering by actor + entity type
- Composite SK ensures sort order is always chronological (newest first when queried descending)

---

## Key Design Principles

### Global Secondary Indexes

**GSI1 (gsi1-index):** `gsi1pk` (HASH) + `gsi1sk` (RANGE)

**Usage:**
- `gsi1pk: DATE#{YYYY-MM-DD}` — Query all meal records for a specific date
- `gsi1pk: WFHPERIOD` — Query all WFH periods sorted by start date
- `gsi1pk: AUDIT_BY_ACTOR#{actorDiscordId}` — Query all changes made by a user

**GSI2 (gsi2-index):** `gsi2pk` (HASH) + `gsi2sk` (RANGE)

**Usage:**
- `gsi2pk: AUDIT_BY_ACTOR_ENTITY#{actorDiscordId}#{entityType}` — Query changes of specific entity type by a user

### Prefix Conventions

- `USER#` — User partition
- `RECORD#` — Meal record sort key
- `DATE#` — Date-based GSI partition
- `SCHEDULE#` — Schedule partition
- `TEAM#` — Team partition
- `WFHPERIOD` — WFH period constant partition
- `AUDIT#` — Audit log partition
- `AUDIT_BY_ACTOR#` — Audit GSI by actor
- `AUDIT_BY_ACTOR_ENTITY#` — Audit GSI by actor + entity type
- `SYSTEM` — System metadata partition

### GSI Overloading

Both GSI1 and GSI2 serve multiple access patterns by using different prefix patterns in the same index attributes. This reduces the total number of GSIs needed while maintaining query efficiency.

### Denormalization Strategy

- Team name stored on user profiles and meal records (avoids joins for common queries)
- Actor name stored on audit logs (historical record, doesn't change if user renamed)
- Team ID stored on meal records (enables headcount grouping without team lookup)

### Sort Key Design

- Timestamps in SK enable chronological ordering
- Date ranges in SK enable 7-day queries with BETWEEN operator
- UUID suffix ensures uniqueness when timestamp alone isn't sufficient

---
