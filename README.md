# DevOps Stage-2: Blue/Green with Nginx Upstreams

## Quick start (local testing)

1. Copy example env and edit if needed:
   ```bash
   cp .env.example .env
   ```

2. Start the stack:
   ```bash
   chmod +x start.sh verify.sh toggle_pool.sh
   ./start.sh
   ```

3. Baseline check:
   ```bash
   curl -i http://localhost:8080/version
   ```

4. Trigger chaos on the active app (blue by default):
   ```bash
   curl -X POST "http://localhost:8081/chaos/start?mode=error"
   ```

5. Run the verification script to observe failover:
   ```bash
   ./verify.sh
   ```

6. Stop chaos:
   ```bash
   curl -X POST "http://localhost:8081/chaos/stop"
   ```