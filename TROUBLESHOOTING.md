╔════════════════════════════════════════════════════════════════════════════════╗
║  Kong Enterprise 3.14 Hybrid Deployment - Troubleshooting Guide                ║
║                          Quick Reference for Common Issues                     ║
╚════════════════════════════════════════════════════════════════════════════════╝


🔍 DIAGNOSTIC COMMANDS (Start here first!)
═══════════════════════════════════════════════════════════════════════════════

# See status of all services
podman-compose ps

# Follow logs in real-time (replace service name)
podman logs -f kong-control-plane
podman logs -f kong-gateway
podman logs -f kong-postgres

# Check system resources (see if any service is out of memory)
podman stats

# Inspect a specific container
podman inspect kong-control-plane | jq '.[0].State'

# List all networks
podman network ls

# Check network connectivity
podman network inspect kong_admin


═══════════════════════════════════════════════════════════════════════════════
ISSUE #1: "License data is invalid" Error
═══════════════════════════════════════════════════════════════════════════════

❌ SYMPTOMS:
   - Kong containers keep restarting
   - Error message: "License data is invalid"
   - In logs: "Invalid license provided"

🔍 DIAGNOSIS:
   podman logs kong-control-plane | grep -i license

✅ SOLUTIONS:

   Problem 1: License JSON is incomplete or corrupted
   ─────────────────────────────────────────────────────
   1. Check if your license JSON is valid:
      - Make sure it has: license, signature, version
      - Should start with {"license": and end with }
   
   2. Verify no characters were cut off:
      echo $KONG_LICENSE_DATA | jq . 2>&1 | head -20
   
   3. Re-copy the license from original email
      - Paste ENTIRE JSON (very long!)
      - Don't use word processors (use plain text editor)

   Problem 2: License expired
   ──────────────────────────
   1. Check expiration date in license:
      echo $KONG_LICENSE_DATA | jq '.license.license_expiration_date'
   
   2. Current timestamp:
      date +%s
   
   3. If expired: Contact Kong for renewal

   Problem 3: Wrong format in .env file
   ────────────────────────────────────
   ❌ WRONG: KONG_LICENSE_DATA={invalid}
   ✅ RIGHT: KONG_LICENSE_DATA='{"license":{...}}'
   
   Single quotes (') are important! Use them.

   Problem 4: Special characters in password breaking license string
   ────────────────────────────────────────────────────────────────
   Try single quotes around entire license:
   KONG_LICENSE_DATA='{"license":{...full license...}}'


═══════════════════════════════════════════════════════════════════════════════
ISSUE #2: "Cannot connect to database" Error
═══════════════════════════════════════════════════════════════════════════════

❌ SYMPTOMS:
   - Control Plane/Data Plane won't start
   - Error: "Error: FATAL: password authentication failed"
   - "Connection refused"

🔍 DIAGNOSIS:
   # Check if postgres is running
   podman ps | grep postgres
   
   # Check postgres logs
   podman logs kong-postgres
   
   # Try connecting directly
   podman exec -it kong-postgres psql -U kong -d kong

✅ SOLUTIONS:

   Problem 1: Wrong password in POSTGRES_PASSWORD
   ──────────────────────────────────────────────
   1. Check what password is set:
      echo $POSTGRES_PASSWORD
   
   2. Verify in compose file it matches:
      grep "POSTGRES_PASSWORD:" kong-enterprise-hybrid-compose.yml
   
   3. If wrong, update .env and restart:
      # Edit .env
      POSTGRES_PASSWORD=correct_password
      
      # Stop and remove containers (not volumes!)
      podman-compose down
      
      # Restart
      podman-compose up -d

   Problem 2: PostgreSQL still initializing
   ────────────────────────────────────────
   Wait 20-30 seconds and try again:
   sleep 30
   podman logs kong-postgres
   
   Look for: "database system is ready to accept connections"

   Problem 3: Volume from previous setup has wrong permissions
   ──────────────────────────────────────────────────────────
   # Remove old volume
   podman volume rm kong_postgres_data
   
   # Start fresh
   podman-compose up -d

   Problem 4: Network connectivity issue
   ────────────────────────────────────
   Verify containers can reach each other:
   
   # From Control Plane, try to reach database
   podman exec kong-control-plane ping kong-postgres
   
   # Should get responses, if not: network issue
   
   Fix: Recreate networks
   podman network rm kong_admin kong_proxy
   podman-compose up -d


═══════════════════════════════════════════════════════════════════════════════
ISSUE #3: Control Plane and Data Plane Not Communicating
═══════════════════════════════════════════════════════════════════════════════

❌ SYMPTOMS:
   - Data Plane can't reach Control Plane
   - Cluster status shows "disconnected"
   - Error: "Connection timeout" in DP logs

🔍 DIAGNOSIS:
   # Check cluster status from CP
   curl -s http://localhost:8001/clustering/status | jq .
   
   # Check DP logs
   podman logs kong-gateway | grep -i cluster

✅ SOLUTIONS:

   Problem 1: Wrong cluster advertise address
   ──────────────────────────────────────────
   In compose file, verify:
   
   kong-control-plane:
     environment:
       KONG_CLUSTER_ADVERTISE: kong-control-plane:8005
       # ↑ Must match the container name!
   
   kong-gateway:
     environment:
       KONG_CLUSTER_CONTROL_PLANE: kong-control-plane:8005
       # ↑ Must match CP container name and port!

   Problem 2: Port 8005 not exposed properly
   ────────────────────────────────────────
   Verify in compose:
   kong-control-plane:
     ports:
       - "8005:8005"  # Required!
   
   Test connectivity:
   podman exec kong-gateway curl -v http://kong-control-plane:8005/

   Problem 3: Network policy blocking traffic
   ──────────────────────────────────────────
   Check both services on same network:
   podman inspect kong-control-plane | grep -A 10 "Networks"
   podman inspect kong-gateway | grep -A 10 "Networks"
   
   Both should show kong_admin network.

   Problem 4: Firewall blocking port 8005
   ──────────────────────────────────────
   Check system firewall:
   # On Linux
   sudo firewall-cmd --list-all
   sudo firewall-cmd --add-port=8005/tcp --permanent


═══════════════════════════════════════════════════════════════════════════════
ISSUE #4: Out of Memory / Resource Issues
═══════════════════════════════════════════════════════════════════════════════

❌ SYMPTOMS:
   - Container crashes randomly
   - Error: "OOMKilled" or "Killed"
   - podman ps shows containers exiting

🔍 DIAGNOSIS:
   # Check system memory
   free -h
   
   # Check Docker stats
   podman stats
   
   # Check container exit code
   podman inspect kong-control-plane | jq '.[0].State.ExitCode'

✅ SOLUTIONS:

   Problem 1: System doesn't have enough RAM
   ────────────────────────────────────────
   Requirements:
   - Total needed: ~2.5GB for all services
   - Free memory: at least 1GB
   
   Check available:
   free -h | grep Mem
   
   If < 1GB free, reduce resource limits in compose:
   
   postgres:
     deploy:
       resources:
         limits:
           memory: 256M    # Reduced from 512M
         reservations:
           memory: 128M    # Reduced from 256M

   Problem 2: One service is memory hungry
   ──────────────────────────────────────
   Identify which:
   podman stats --no-stream kong-postgres kong-control-plane kong-gateway
   
   If Control Plane using too much:
   - Check for misconfigured plugins
   - Reduce KONG_LOG_LEVEL from "notice" to "warn"
   - Restart service

   Problem 3: Database bloated with old data
   ────────────────────────────────────────
   Check database size:
   podman exec kong-postgres \
     du -sh /var/lib/postgresql/data
   
   Clean up old data:
   podman exec -it kong-postgres psql -U kong -d kong -c \
     "DELETE FROM logs WHERE created_at < NOW() - INTERVAL '30 days';"


═══════════════════════════════════════════════════════════════════════════════
ISSUE #5: Port Already in Use
═══════════════════════════════════════════════════════════════════════════════

❌ SYMPTOMS:
   - Error: "bind: address already in use"
   - Can't start containers
   - Error mentions port 8001, 8000, etc.

🔍 DIAGNOSIS:
   # Find what's using port 8001
   podman ps | grep 8001
   lsof -i :8001          # On Linux
   netstat -an | grep 8001

✅ SOLUTIONS:

   Problem 1: Port used by another Podman container
   ────────────────────────────────────────────────
   1. Find which container:
      podman ps | grep -E "8001|8000|8002"
   
   2. Stop conflicting container:
      podman stop container_name
      podman rm container_name
   
   3. Retry:
      podman-compose up -d

   Problem 2: Port used by other application
   ─────────────────────────────────────────
   Option A: Stop the other application
   Option B: Use different port in compose file:
   
   ports:
     - "9001:8001"    # External port changed to 9001
     - "8000:8000"
     - "8002:8002"
   
   Then access Admin API at: localhost:9001 (not 8001)

   Problem 3: Previous Kong setup still running
   ────────────────────────────────────────────
   Cleanup old setup:
   podman-compose down -v
   podman volume prune
   podman network prune


═══════════════════════════════════════════════════════════════════════════════
ISSUE #6: Can't Access Kong Manager UI
═══════════════════════════════════════════════════════════════════════════════

❌ SYMPTOMS:
   - Browser shows "Connection refused"
   - http://localhost:8002 doesn't load
   - Blank page or timeout

🔍 DIAGNOSIS:
   # Check if port 8002 is listening
   curl -v http://localhost:8002/
   
   # Check if service is running
   podman ps | grep kong-control-plane
   
   # Check if port is mapped
   podman port kong-control-plane

✅ SOLUTIONS:

   Problem 1: Control Plane not fully started yet
   ──────────────────────────────────────────────
   Wait 30-60 seconds and try again:
   sleep 30
   curl http://localhost:8002/
   
   If still not working, check logs:
   podman logs kong-control-plane | tail -20

   Problem 2: Port 8002 not exposed in compose file
   ────────────────────────────────────────────────
   Verify in compose:
   kong-control-plane:
     ports:
       - "8002:8002"  # Required!
   
   If missing, add it and restart:
   podman-compose down
   podman-compose up -d

   Problem 3: Wrong login credentials
   ──────────────────────────────────
   Default credentials:
   Username: kong_admin
   Password: kong_admin
   
   If you changed them, use your new values.
   
   Reset to defaults:
   # Edit .env
   KONG_ADMIN_USER=kong_admin
   KONG_ADMIN_PASSWORD=kong_admin
   
   # Restart
   podman-compose restart kong-control-plane

   Problem 4: Browser cache issue
   ──────────────────────────────
   Try:
   1. Hard refresh: Ctrl+Shift+R (Cmd+Shift+R on Mac)
   2. Clear cookies: DevTools → Application → Cookies
   3. Try incognito window
   4. Try different browser


═══════════════════════════════════════════════════════════════════════════════
ISSUE #7: Data Plane Not Processing API Requests
═══════════════════════════════════════════════════════════════════════════════

❌ SYMPTOMS:
   - HTTP calls to port 8000 timeout or return 503
   - Error: "503 Service Unavailable"
   - Gateway doesn't route traffic

🔍 DIAGNOSIS:
   # Test DP is alive
   curl -v http://localhost:8000/status
   
   # Check DP logs
   podman logs kong-gateway
   
   # Check cluster sync
   curl -s http://localhost:8001/clustering/status | jq '.data[0]'

✅ SOLUTIONS:

   Problem 1: Data Plane not synchronized with Control Plane
   ─────────────────────────────────────────────────────────
   1. Check sync status:
      curl -s http://localhost:8001/clustering/status | jq .
   
   2. Look for:
      "status": "connected"    ✓ Good
      "status": "unknown"      ✗ Bad
   
   3. Restart DP if disconnected:
      podman-compose restart kong-gateway

   Problem 2: No services/routes configured in CP
   ──────────────────────────────────────────────
   Add a test service:
   curl -X POST http://localhost:8001/services \
     -H "Content-Type: application/json" \
     -d '{"name":"test","url":"http://httpbin.org"}'
   
   Add a route:
   curl -X POST http://localhost:8001/services/test/routes \
     -H "Content-Type: application/json" \
     -d '{"paths":["/test"]}'
   
   Test route:
   curl -i http://localhost:8000/test

   Problem 3: Data Plane crashed or unhealthy
   ──────────────────────────────────────────
   Check health:
   podman logs kong-gateway | grep -i error
   
   Restart:
   podman-compose restart kong-gateway
   
   Check again:
   curl -s http://localhost:8000/status


═══════════════════════════════════════════════════════════════════════════════
ISSUE #8: High CPU or Disk Usage
═══════════════════════════════════════════════════════════════════════════════

❌ SYMPTOMS:
   - System running slow
   - CPU at 100%
   - Disk full

🔍 DIAGNOSIS:
   # Check CPU/Memory
   podman stats
   
   # Check disk usage
   df -h
   
   # Check log size
   podman logs kong-control-plane | wc -l

✅ SOLUTIONS:

   Problem 1: Kong services logging too much
   ────────────────────────────────────────
   Current logging in compose uses json-file driver with:
   - max-size: 10m
   - max-file: 3
   
   This limits to 30MB per service (fine).
   
   Check actual size:
   du -h /var/lib/containers/storage/containers/*/
   
   If issue persists, reduce log level:
   
   In compose file, change:
   KONG_LOG_LEVEL: notice  →  KONG_LOG_LEVEL: warn
   
   Restart services.

   Problem 2: Database got very large
   ──────────────────────────────────
   Check size:
   podman exec kong-postgres \
     du -sh /var/lib/postgresql/data
   
   If > 1GB with minimal data, may have old data.
   
   Cleanup:
   1. Backup database:
      podman exec kong-postgres pg_dump -U kong kong > backup.sql
   
   2. Vacuum database:
      podman exec -it kong-postgres psql -U kong -d kong -c "VACUUM FULL;"

   Problem 3: Excessive request logging
   ───────────────────────────────────
   Disable access logs:
   In compose, add:
   KONG_PROXY_ACCESS_LOG: /dev/null
   KONG_ADMIN_ACCESS_LOG: /dev/null


═══════════════════════════════════════════════════════════════════════════════
QUICK RECOVERY PROCEDURE (When everything is broken)
═══════════════════════════════════════════════════════════════════════════════

1. Stop everything:
   podman-compose down

2. Keep database volume (we want to preserve config):
   podman volume ls | grep kong_postgres_data

3. Start fresh:
   podman-compose up -d

4. Wait for services:
   sleep 30

5. Verify:
   podman-compose ps
   curl -s http://localhost:8001/status | jq .
   curl -s http://localhost:8000/status | jq .

6. If still broken, nuclear option:
   # DELETE EVERYTHING (will lose all Kong configuration!)
   podman-compose down -v
   podman volume rm kong_postgres_data
   podman network rm kong_admin kong_proxy
   podman-compose up -d


═══════════════════════════════════════════════════════════════════════════════
USEFUL DEBUGGING COMMANDS
═══════════════════════════════════════════════════════════════════════════════

# View environment variables of running container
podman exec kong-control-plane env | grep KONG

# Connect to running database and query
podman exec -it kong-postgres psql -U kong -d kong -c "SELECT * FROM services LIMIT 5;"

# Monitor live logs (all containers)
podman-compose logs -f

# Export logs to file for analysis
podman logs kong-control-plane > cp-debug.log 2>&1
podman logs kong-gateway > dp-debug.log 2>&1

# Check DNS resolution inside container
podman exec kong-gateway nslookup kong-control-plane

# Simulate API request from DP to test configuration
podman exec kong-gateway curl -v http://kong-control-plane:8005/clustering/status

# Check disk space inside containers
podman exec kong-postgres df -h


═══════════════════════════════════════════════════════════════════════════════
When All Else Fails
═══════════════════════════════════════════════════════════════════════════════

Collect information for Kong support:

1. Save all logs:
   podman logs kong-control-plane > logs-cp.txt 2>&1
   podman logs kong-gateway > logs-dp.txt 2>&1
   podman logs kong-postgres > logs-db.txt 2>&1

2. Capture container status:
   podman ps -a > container-status.txt
   podman inspect kong-control-plane > cp-inspect.json

3. Get compose file info:
   cat kong-enterprise-hybrid-compose.yml > compose-config.txt

4. System information:
   podman version > version.txt
   uname -a >> version.txt

5. Contact Kong Support:
   Email: support@konghq.com
   Include: logs-*.txt, cp-inspect.json, version.txt
   
   Tell them:
   - What error you see
   - What you've already tried
   - Your environment (Podman version, OS, resources)
   - Attach all collected files


═══════════════════════════════════════════════════════════════════════════════
End of Troubleshooting Guide
═══════════════════════════════════════════════════════════════════════════════
