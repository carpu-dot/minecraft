# Carpuncle Agent — Phase 3 Autonomous Approval Policy

**Status:** ACTIVE  
**Effective Date:** 2026-08-28T12:30:00Z  
**Policy Version:** 1.0  
**Owner:** carpu-dot  
**Approval Authority:** carpu-dot (GitHub Admin on carpu-dot/minecraft)

---

## 1. Scope

This policy authorizes the Carpuncle Server Agent to autonomously execute **Phase 3 operations** on the Minecraft server (`178.63.80.31`/Purpur) without human intervention, subject to the conditions and safeguards outlined below.

Phase 3 operations include:
- Plugin installation, updates, and removal
- Datapack deployment and rotation
- Server configuration changes (non-breaking)
- Loot table and gameplay feature updates
- Seasonal event activation
- Automatic rollback on detection of critical errors

---

## 2. Authorized Operations

### 2.1 Plugin Management
- **Install:** Download from approved sources (GitHub releases, SpigotMC, PaperMC), verify signature/hash, test in staging, deploy to plugins/ folder
- **Update:** Check for updates to installed plugins, run pre-update backup, apply update, verify server startup, auto-rollback if critical error detected
- **Remove:** Uninstall disabled plugins, clean up configuration orphans, verify no dangling dependencies

**Conditions:**
- Plugin must be on the `approved-plugins.yaml` allowlist
- SHA256 hash must match published release hash
- Server must remain accessible (port 25565 up) after deployment
- If post-deployment error rate exceeds threshold (see 3.1), trigger auto-rollback

### 2.2 Datapack Deployment
- Deploy new datapacks to `world/datapacks/`
- Activate/deactivate datapacks via `/datapack` commands
- Rotate seasonal datapacks on schedule (defined in `seasonal-calendar.yaml`)
- Validate datapack JSON syntax before deployment

**Conditions:**
- Datapack must be sourced from approved repository or validated against `approved-datapacks.yaml`
- Agent must test datapack in a non-production world first (if staging available)
- After activation, agent must monitor error logs for 5 minutes; if critical errors detected, auto-disable datapack

### 2.3 Configuration Changes
- Update `server.properties` and `purpur.yml` for non-breaking changes (e.g., difficulty, PvP toggle, spawn-protection)
- Update plugin configs (e.g., BlueMap render distance, Geyser settings)
- Apply performance tuning (e.g., view-distance, simulation-distance, entity-tracking-range)

**Conditions:**
- Configuration change must be defined in `approved-configs.yaml` with rationale
- Agent must create backup before applying any change
- Server must restart cleanly after change; if restart fails, auto-rollback to previous config
- No changes to authentication, secret keys, or security-critical settings without explicit owner approval

### 2.4 Gameplay Features & Seasonal Events
- Create/update custom loot tables, advancement trees, recipe books
- Schedule and activate seasonal events (Halloween, Christmas, summer camps, etc.)
- Deploy temporary world modifications (event-specific datapacks, structures)

**Conditions:**
- Feature must be documented in `approved-features.yaml` with clear purpose and rollback procedure
- Event must have defined start/end date and auto-cleanup logic
- Agent must log all feature deployments in `deployments.log`

### 2.5 Automatic Rollback
- Rollback any Phase 3 change if:
  - Server fails to start after deployment
  - Critical error count exceeds threshold within 5 minutes of deployment
  - Agent detects incompatibility (e.g., plugin version mismatch with server version)
  - Server becomes inaccessible (port 25565 down)

**Rollback Procedure:**
1. Stop server
2. Restore backup from pre-change snapshot
3. Restart server
4. Verify server accessibility
5. Log rollback event with timestamp and reason in `rollback.log`
6. Notify owner (via GitHub issue or webhook) of rollback

---

## 3. Safeguards & Monitoring

### 3.1 Error Thresholds
- **Critical Error:** Any `[SEVERE]` log entry or Java exception in latest.log
- **Warning Error:** Any `[WARNING]` or `[ERROR]` entry in latest.log
- **Threshold:** If ≥ 3 critical errors within 5 minutes post-deployment → auto-rollback
- **Threshold:** If ≥ 10 warning errors within 5 minutes post-deployment → alert owner, do not auto-rollback

### 3.2 Backup Policy
- **Frequency:** Full backup before each Phase 3 operation
- **Retention:** Keep last 10 backups; delete older backups automatically
- **Location:** `/srv/minecraft/backups/`
- **Format:** `backup-YYYYMMDD-HHMMSS-{operation}.tar.gz`
- **Verification:** Agent must verify backup integrity (tar test) before proceeding with deployment

### 3.3 Staging Environment
- If available, agent must test plugin updates and datapacks in staging world before production deployment
- Staging world is an exact copy of production; agent validates behavior before promoting to main world

### 3.4 Audit Logging
All Phase 3 operations must be logged to `deployments.log` with:
- Timestamp (UTC)
- Operation type (install, update, datapack, config, rollback)
- Asset name and version
- Pre-deployment backup location
- Post-deployment result (success, warning, critical error, rolled back)
- Any human intervention required

### 3.5 Health Check
- After every Phase 3 operation, agent runs `server_status` action
- Verify: service state (active), MainPID (valid), Minecraft ready (true), latest.log (no critical errors within last 5 min)
- If health check fails, trigger rollback

---

## 4. Allowlists & Configuration Files

The agent reads and enforces these allowlist files from the repository root:

### 4.1 `approved-plugins.yaml`
```yaml
plugins:
  - name: BlueMap
    source: github:BlueMap-Minecraft/BlueMap
    min_version: "5.0"
    max_version: "6.0"
    sha256_override: null
  - name: Geyser
    source: github:GeyserMC/Geyser
    min_version: "2.0"
    max_version: "3.0"
    sha256_override: null
  # ... more plugins
```

### 4.2 `approved-datapacks.yaml`
```yaml
datapacks:
  - name: seasonal-summer
    source: github:carpu-dot/minecraft-datapacks
    active_from: "2026-06-21"
    active_until: "2026-09-20"
    auto_cleanup: true
  # ... more datapacks
```

### 4.3 `approved-configs.yaml`
```yaml
configs:
  - file: "server.properties"
    properties:
      difficulty: [ "easy", "normal", "hard" ]
      pvp: [ true, false ]
      spawn-protection: [ 0, 50, 100, 200 ]
  - file: "purpur.yml"
    properties:
      gameplay.flying-squid.enable: [ true, false ]
  # ... more configs
```

### 4.4 `approved-features.yaml`
```yaml
features:
  - name: halloween-event-2026
    type: seasonal-event
    datapack_source: github:carpu-dot/minecraft-events
    start_date: "2026-10-01"
    end_date: "2026-10-31"
    auto_cleanup: true
  # ... more features
```

### 4.5 `seasonal-calendar.yaml`
```yaml
seasons:
  - name: summer
    start: "06-21"
    end: "09-20"
    datapacks: [ seasonal-summer ]
  - name: autumn
    start: "09-21"
    end: "12-20"
    datapacks: [ seasonal-autumn ]
  # ... more seasons
```

---

## 5. Rollback & Emergency Stop

### 5.1 Automatic Rollback
- Triggered by error threshold breach or server accessibility loss (see 3.1, 3.5)
- Agent executes rollback automatically without waiting for owner approval

### 5.2 Emergency Stop
- **Manual Override:** Owner can create a GitHub issue with label `emergency-stop` in `carpu-dot/minecraft`
- **Automated Stop:** If agent detects persistent critical errors (5+ consecutive rollbacks in 1 hour), agent stops and creates issue `agent-halted`
- **Resume:** Agent can only resume Phase 3 operations after owner updates issue with label `resume-approved`

---

## 6. Notification & Reporting

### 6.1 Notifications (Webhook)
Agent sends webhook notifications to a configured endpoint for:
- Successful Phase 3 deployment (info level)
- Rollback events (warning level)
- Agent halt (critical level)
- Threshold breaches (warning level)

### 6.2 Logs & Reports
- **deployments.log:** All Phase 3 operations with results
- **rollback.log:** All rollback events with reason
- **server_status.json:** Periodic health check snapshots (hourly)
- All logs stored in `/srv/minecraft/logs/carpuncle/`

---

## 7. Policy Amendments

This policy is effective immediately upon merge to `main` branch of `carpu-dot/minecraft`.

**To modify this policy:**
1. Create a pull request with changes
2. Owner must review and approve
3. Merge to `main`
4. Agent re-reads policy on next scheduled check (every 30 minutes) or immediately on webhook notification

**Policy Review Schedule:** Every 90 days or as needed

---

## 8. Acknowledgment

**Approved By:** carpu-dot  
**Date:** 2026-08-28  
**Signature:** carpu-dot (GitHub account verified)

I acknowledge and accept this Phase 3 Autonomous Approval Policy and authorize the Carpuncle Server Agent to execute Phase 3 operations as defined above, subject to all safeguards, error thresholds, and rollback procedures outlined in this document.

---

**End of Policy**
