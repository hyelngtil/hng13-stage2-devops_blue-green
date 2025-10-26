# Blue/Green Deployment with Nginx Auto-Failover

A Blue/Green deployment system using Nginx upstream load balancing with automatic failover capabilities. This implementation ensures zero-downtime deployments and graceful handling of service failures.

## Architecture Overview

```
                    ┌─────────────┐
                    │   Client    │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │    Nginx    │ :8080 (Public)
                    │  (Primary/  │
                    │   Backup)   │
                    └──┬───────┬──┘
                       │       │
          ┌────────────┘       └────────────┐
          │                                  │
    ┌─────▼─────┐                     ┌─────▼─────┐
    │ App Blue  │ :8081               │ App Green │ :8082
    │ (Primary) │                     │  (Backup) │
    └───────────┘                     └───────────┘
```

**Key Features:**
- **Automatic Failover**: Nginx detects failures and switches to backup within 2-3 seconds
- **Zero Failed Requests**: Retry mechanism ensures clients always receive 200 OK
- **Manual Toggle**: Switch active pool on-demand for deployments
- **Health-Based Routing**: Uses `max_fails` and `fail_timeout` for intelligent routing
- **Header Preservation**: Maintains application headers (X-App-Pool, X-Release-Id)

## Prerequisites
- **Docker** >= 20.10.0
- **Docker Compose** (plugin version)
- **Bash** shell
- **curl** (for testing)
- **envsubst** (usually included with gettext-base)

## Quick Start (Local Testing)

### 1. Initial Setup
```bash
# Clone the repository
git clone <your-repo-url>
cd <repo-directory>

# Copy and configure environment
cp .env.example .env

# Make scripts executable
chmod +x start.sh verify.sh toggle_pool.sh
```

### 2. Start the Stack
```bash
./start.sh
```
**Expected Output:**
```
Rendered nginx/default.conf with PRIMARY=app_blue, SECONDARY=app_green, PORT=8080
Starting docker compose...
[+] Running 3/3
 ✔ Container app_blue   Started
 ✔ Container app_green  Started  
 ✔ Container nginx      Started
Waiting 1s for nginx to start...
Done. Nginx on http://localhost:8080
```

### 3. Verify Baseline (Blue Active)
```bash
curl -i http://localhost:8080/version
```
**Expected Response:**
```http
HTTP/1.1 200 OK
Server: nginx/1.24.0
X-App-Pool: blue
X-Release-Id: release-blue-001
Content-Type: application/json

{"version":"1.0.0","pool":"blue","release":"release-blue-001"}
```

### 4. Test Automatic Failover
**Step A: Trigger Chaos on Blue**
```bash
curl -X POST "http://localhost:8081/chaos/start?mode=error"
```

**Step B: Run Verification Script**
```bash
./verify.sh
```
**Expected Behavior:**
- First 1-2 requests may show Blue (200 OK)
- Nginx detects Blue failures (500 errors)
- Automatically switches to Green within 2-3 seconds
- All subsequent requests show Green (200 OK)
- **Zero 5xx errors to clients** (Nginx retries to backup)

**Sample Output:**
```
1: HTTP/1.1 200 OK  X-App-Pool: blue  X-Release-Id: release-blue-001
2: HTTP/1.1 200 OK  X-App-Pool: blue  X-Release-Id: release-green-001
3: HTTP/1.1 200 OK  X-App-Pool: green X-Release-Id: release-green-001
4: HTTP/1.1 200 OK  X-App-Pool: green X-Release-Id: release-green-001
...
20: HTTP/1.1 200 OK X-App-Pool: green X-Release-Id: release-green-001
```

**Step C: Stop Chaos**
```bash
curl -X POST "http://localhost:8081/chaos/stop"
```

### 5. Manual Pool Toggle (Blue ↔ Green)
```bash
# Switch active pool (e.g., from blue to green)
./toggle_pool.sh

# Verify the switch
curl -i http://localhost:8080/version
```
**Use Cases:**
- Deploying new application versions
- Planned maintenance windows
- Blue/Green release strategy

## Configuration files
- Environment Variables (.env)
- Nginx Failover Configuration (nginx/nginx.conf.template)

## Testing & Verification
### Available Endpoints
**Via Nginx (Public):**
- `GET http://localhost:8080/version` - Version info with routing headers
- `GET http://localhost:8080/healthz` - Health check endpoint

**Direct Access (For Chaos Testing):**
- `GET http://localhost:8081/version` - Blue app direct
- `POST http://localhost:8081/chaos/start?mode=error` - Trigger 500 errors
- `POST http://localhost:8081/chaos/start?mode=timeout` - Trigger timeouts
- `POST http://localhost:8081/chaos/stop` - Stop chaos
- `GET http://localhost:8082/version` - Green app direct

### Chaos Modes
1. **Error Mode**: Returns HTTP 500 errors
   ```bash
   curl -X POST "http://localhost:8081/chaos/start?mode=error"
   ```

2. **Timeout Mode**: Delays responses beyond timeout threshold
   ```bash
   curl -X POST "http://localhost:8081/chaos/start?mode=timeout"
   ```

### Load Testing
```bash
# Sequential requests (observe failover)
for i in {1..50}; do
  curl -s http://localhost:8080/version | jq -r '.pool'
  sleep 0.1
done
```

### Validation Checklist
- [ ] All containers running: `docker compose ps`
- [ ] Blue responds: `curl http://localhost:8081/version` → 200 OK
- [ ] Green responds: `curl http://localhost:8082/version` → 200 OK
- [ ] Nginx routes correctly: `curl http://localhost:8080/version` → 200 OK
- [ ] Headers preserved: `X-App-Pool` and `X-Release-Id` present
- [ ] Failover works: Zero 5xx during chaos
- [ ] Manual toggle switches pool successfully
- [ ] CI pipeline passes all checks

## Success Metrics
### Failover Performance
- **Detection Time**: < 2 seconds
- **Switch Time**: < 1 second (first retry)
- **Total Failover**: < 3 seconds
- **Failed Requests**: 0 (100% success rate)

### Response Distribution During Chaos
- **Before Failover**: 100% Blue
- **During Failover**: 95-98% Green (within 10 seconds)
- **After Failover**: 100% Green

## Monitoring & Debugging
### Container Status
```bash
# Check all containers
docker compose ps

# View logs
docker compose logs -f
docker compose logs nginx
docker compose logs app_blue
docker compose logs app_green

## CI/CD Pipeline
### GitHub Actions Workflow
The `.github/workflows/ci.yml` automatically:
1. ✅ Starts the full stack
2. ✅ Verifies baseline (Blue active)
3. ✅ Triggers chaos on Blue
4. ✅ Validates automatic failover
5. ✅ Ensures zero 5xx errors
6. ✅ Confirms Green takes over
7. ✅ Stops chaos and cleans up

**Trigger Events:**
- Push to any branch
- Pull requests

**Success Criteria:**
- All 20 requests return HTTP 200
- Headers switch from Blue to Green after chaos
- No failed requests during failover.

## Project Structure
```
.
├── .env.example              # Environment template
├── .gitignore               # Git ignore rules
├── docker-compose.yml       # Container orchestration
├── README.md                # This file
├── start.sh                 # Start stack with config rendering
├── toggle_pool.sh           # Manual pool switching
├── verify.sh                # Automated failover testing
├── nginx/
│   └── nginx.conf.template  # Nginx configuration template
└── .github/
    └── workflows/
        └── ci.yml           # GitHub Actions CI pipeline
```

## Failure Scenarios Handled
| Scenario | Detection | Recovery | Downtime |
|----------|-----------|----------|----------|
| App crashes | Health check fails | Switch to backup | < 3s |
| App hangs | Read timeout | Retry to backup | < 5s |
| App returns 500 | HTTP status check | Retry to backup | < 1s |
| Network partition | Connection timeout | Switch to backup | < 2s |
| Planned maintenance | Manual toggle | Graceful switch | 0s |

## Cleanup
```bash
# Stop all containers
docker compose down

# Remove volumes and networks
docker compose down -v

# Remove generated configs
rm -f nginx/default.conf

# Full cleanup (removes images)
docker compose down -v --rmi all
```

## ✅ Validation Results
Expected CI/CD output when all checks pass:
```
✓ Containers started successfully
✓ Baseline check: Blue active (200 OK)
✓ Chaos triggered on Blue
✓ 20/20 requests successful (100% pass rate)
✓ Failover detected: Green became active
✓ Headers preserved correctly
✓ Zero failed requests during chaos
✓ All tests passed!