╔════════════════════════════════════════════════════════════════════════════════╗
║     Best Practices Applied to Kong Enterprise Hybrid Deployment                ║
║              Docker Compose File - Detailed Explanation                        ║
╚════════════════════════════════════════════════════════════════════════════════╝

This document explains EVERY best practice decision made in the compose file.
Use this to understand WHY things are done this way.


1️⃣  COMPOSE FILE VERSION & STRUCTURE
═══════════════════════════════════════════════════════════════════════════════

❌ WRONG:
   version: '2'        # Old version, missing features

✅ RIGHT (What we did):
   version: '3.8'      # Latest stable version supporting all features

WHY:
  - Version 3.8 supports resource limits (limits/reservations)
  - Supports named volumes with better control
  - Supports healthchecks (critical for startup order)
  - Better logging options
  - Better secret management


2️⃣  SERVICE DEPENDENCIES & STARTUP ORDER
═══════════════════════════════════════════════════════════════════════════════

❌ WRONG:
   kong-control-plane:
     image: kong/kong-gateway-enterprise:3.14
     # No depends_on, services start randomly
     # Control Plane might start before database is ready

✅ RIGHT (What we did):
   kong-control-plane:
     image: kong/kong-gateway-enterprise:3.14
     depends_on:
       postgres:
         condition: service_healthy    # Wait for database to be HEALTHY
     
   kong-gateway:
     image: kong/kong-gateway-enterprise:3.14
     depends_on:
       postgres:
         condition: service_healthy
       kong-control-plane:
         condition: service_healthy    # Wait for CP to be ready

WHY:
  - "condition: service_healthy" waits for the healthcheck to pass
  - Prevents startup race conditions
  - Services won't fail due to missing dependencies
  - Guaranteed proper startup order


3️⃣  HEALTH CHECKS
═══════════════════════════════════════════════════════════════════════════════

❌ WRONG (No health checks):
   postgres:
     image: postgres:15-alpine
     # No way to know if database is ready

✅ RIGHT (What we did):
   postgres:
     healthcheck:
       test: ["CMD", "pg_isready", "-U", "kong", "-d", "kong"]
       interval: 10s        # Check every 10 seconds
       timeout: 5s          # Timeout after 5 seconds
       retries: 5           # Fail after 5 failed checks
       start_period: 20s    # Allow 20s startup time before checking

   kong-control-plane:
     healthcheck:
       test: ["CMD", "kong", "health"]
       interval: 10s
       timeout: 5s
       retries: 5
       start_period: 30s    # CP needs more time (30s) than DB (20s)

WHY:
  - "start_period" allows services time to initialize
  - "interval" checks frequently enough to catch failures
  - "retries" prevents false negatives from temporary issues
  - Different start_periods for different services reflect reality
  - Other services can now safely depend_on with "service_healthy"


4️⃣  RESOURCE LIMITS
═══════════════════════════════════════════════════════════════════════════════

❌ WRONG (No resource limits):
   postgres:
     image: postgres:15-alpine
     # Container can consume all system resources, crash others

✅ RIGHT (What we did):
   postgres:
     deploy:
       resources:
         limits:
           cpus: '1'           # Max 1 CPU core
           memory: 512M        # Max 512MB RAM
         reservations:
           cpus: '0.5'         # Reserve 0.5 CPU
           memory: 256M        # Reserve 256MB RAM
   
   kong-control-plane:
     deploy:
       resources:
         limits:
           cpus: '2'           # More CPU for management tasks
           memory: 1G
         reservations:
           cpus: '1'
           memory: 512M

WHY:
  - "limits" prevent runaway processes from crashing entire system
  - "reservations" guarantee minimum resources for each service
  - Different resource allocation for different roles:
    * DB: Balanced (1 CPU, 512MB)
    * CP: Generous (2 CPU, 1GB) - handles admin API, UI, cluster sync
    * DP: Generous (2 CPU, 1GB) - processes all API requests
  - Local development: 4 CPU total, 2.5GB RAM needed
  - Prevent resource starvation
  - Easier to scale (just change numbers)


5️⃣  LOGGING CONFIGURATION
═══════════════════════════════════════════════════════════════════════════════

❌ WRONG:
   postgres:
     image: postgres:15-alpine
     # Logs go to Podman/Docker daemon, no size limit, fills disk

✅ RIGHT (What we did):
   postgres:
     logging:
       driver: "json-file"      # Structured JSON logging
       options:
         max-size: "10m"        # Max 10MB per log file
         max-file: "3"          # Keep only 3 rotated files

WHY:
  - "json-file" driver is standard, parseable, and lightweight
  - "max-size: 10m" prevents logs from filling up disk
  - "max-file: 3" means only 30MB max per service (3 files × 10MB)
  - Logs are automatically rotated and old logs deleted
  - Easier to monitor without overwhelming storage
  - Same logging for all services for consistency


6️⃣  ENVIRONMENT VARIABLES - CONFIGURATION MANAGEMENT
═══════════════════════════════════════════════════════════════════════════════

❌ WRONG:
   postgres:
     environment:
       POSTGRES_USER: kong
       POSTGRES_PASSWORD: kong         # Hardcoded default password!

✅ RIGHT (What we did):
   postgres:
     environment:
       POSTGRES_USER: kong
       POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-kong_secure_password}
       #                  ↑ Load from .env file, fallback to default

   kong-control-plane:
     environment:
       KONG_LICENSE_DATA: ${KONG_LICENSE_DATA:-'{"license":{...}}'}
       #                  ↑ Load from .env file for security

WHY:
  - "${VAR_NAME}" pulls from .env file (secure, not in source control)
  - ":-default_value" provides fallback for development
  - Passwords and licenses not hardcoded in compose file
  - Can override at runtime: 
    POSTGRES_PASSWORD=mypass podman-compose up
  - Different values per environment (dev/staging/prod)


7️⃣  HYBRID MODE CONFIGURATION
═══════════════════════════════════════════════════════════════════════════════

❌ WRONG (Both acting as control plane):
   kong-control-plane:
     environment:
       KONG_ROLE: control_plane
       # Admin API, proxy, everything mixed

   kong-gateway:
     environment:
       KONG_ROLE: control_plane  # Wrong! Should be data_plane

✅ RIGHT (What we did):
   kong-control-plane:
     environment:
       KONG_ROLE: control_plane
       KONG_ADMIN_LISTEN: 0.0.0.0:8001
       KONG_ADMIN_GUI_LISTEN: 0.0.0.0:8002
       KONG_CLUSTER_LISTEN: 0.0.0.0:8005
       KONG_CLUSTER_ADVERTISE: kong-control-plane:8005

   kong-gateway:
     environment:
       KONG_ROLE: data_plane
       KONG_PROXY_LISTEN: 0.0.0.0:8000
       KONG_PROXY_LISTEN_SSL: 0.0.0.0:8443
       KONG_CLUSTER_CONTROL_PLANE: kong-control-plane:8005
       KONG_CLUSTER_TELEMETRY_ENDPOINT: kong-control-plane:8006

WHY:
  - KONG_ROLE determines what the service does
  - Control Plane handles management (admin API, UI)
  - Data Plane handles proxying (incoming API requests)
  - KONG_CLUSTER_ADVERTISE: CP tells DP how to reach it
  - KONG_CLUSTER_CONTROL_PLANE: DP knows where CP is
  - Separation of concerns = easier to scale


8️⃣  PORT MAPPING - SECURITY & CLARITY
═══════════════════════════════════════════════════════════════════════════════

❌ WRONG:
   kong-control-plane:
     ports:
       - "8001:8001"
       - "8002:8002"
       - "8005:8005"
     # No comments, unclear what each port does

✅ RIGHT (What we did):
   kong-control-plane:
     ports:
       # Admin API (configuration management)
       - "8001:8001"
       # Admin GUI / Manager (web dashboard)
       - "8002:8002"
       # Cluster communication port (CP-DP sync)
       - "8005:8005"

   kong-gateway:
     ports:
       # HTTP proxy port
       - "8000:8000"
       # HTTPS proxy port
       - "8443:8443"

WHY:
  - Comments explain the purpose of each port
  - Port mapping is intentional, not random
  - Cluster ports (8005, 8006) on CP only
  - Proxy ports (8000, 8443) on DP only
  - External API clients → DP (8000, 8443)
  - Admin clients → CP (8001, 8002)
  - Multiple DP instances: use 8010, 8011, etc.


9️⃣  NETWORKING - ISOLATION & COMMUNICATION
═══════════════════════════════════════════════════════════════════════════════

❌ WRONG:
   # Using default 'bridge' network, all services talk to all services
   # No isolation, security risk

✅ RIGHT (What we did):
   networks:
     kong_admin:
       driver: bridge
       ipam:
         config:
           - subnet: 172.20.0.0/16
     
     kong_proxy:
       driver: bridge
       ipam:
         config:
           - subnet: 172.21.0.0/16

   postgres:
     networks:
       - kong_admin

   kong-control-plane:
     networks:
       - kong_admin

   kong-gateway:
     networks:
       - kong_admin        # For CP-DP communication & DB access
       - kong_proxy        # For external API client traffic

WHY:
  - kong_admin: Private network for internal communication only
    - CP talks to DP
    - Both talk to PostgreSQL
    - NOT accessible from outside
  
  - kong_proxy: Public network for external traffic
    - External API clients connect here
    - Database is NOT on this network
    - Prevents accidental database exposure
  
  - Network isolation prevents:
    - Accidental exposure of database ports
    - External clients accessing admin API
    - Cross-contamination of concerns


🔟 VOLUMES - PERSISTENT DATA
═══════════════════════════════════════════════════════════════════════════════

❌ WRONG:
   postgres:
     # No volume! Data lost when container stops

✅ RIGHT (What we did):
   postgres:
     volumes:
       - kong_postgres_data:/var/lib/postgresql/data

   volumes:
     kong_postgres_data:
       driver: local

WHY:
  - Named volume "kong_postgres_data" stores database files
  - Persists across container restarts
  - Data survives podman-compose down
  - Easier to back up (just copy the volume)
  - Can use with external storage drivers (NFS, Ceph, etc.)
  - To delete: podman volume rm kong_postgres_data


1️⃣1️⃣  SECURITY - LICENSE KEY HANDLING
═══════════════════════════════════════════════════════════════════════════════

❌ WRONG:
   kong-control-plane:
     environment:
       KONG_LICENSE_DATA: '{"license":{"payload":{...actual_license...}}}'
       # License hardcoded in file, exposed in version control!

✅ RIGHT (What we did):
   kong-control-plane:
     environment:
       KONG_LICENSE_DATA: ${KONG_LICENSE_DATA:-'{"license":{...placeholder...}}'}

   # .env file (NOT in git):
   KONG_LICENSE_DATA='{"license":{"payload":{...actual_license...}}}'

WHY:
  - License key is NEVER hardcoded in compose file
  - Lives in .env file which is in .gitignore
  - Can be loaded from environment at runtime
  - Production uses secure secret management (Docker Secrets, vault, etc.)
  - Prevents accidental key exposure in source control


1️⃣2️⃣  DOCUMENTATION & COMMENTS
═══════════════════════════════════════════════════════════════════════════════

❌ WRONG:
   version: '3.8'
   services:
     postgres:
       image: postgres:15-alpine
       # No explanation of what this is for

✅ RIGHT (What we did):
   # Kong Enterprise 3.14 - Hybrid Mode Deployment
   # Topology: Separated Control Plane, Data Plane (Gateway), and PostgreSQL Database
   # Services:
   #   - postgres: Shared database for both CP and DP
   #   - kong-control-plane: Management plane (admin API, Manager UI)
   #   - kong-gateway: Data plane (API proxy)

WHY:
  - File header explains the entire setup
  - Section comments clarify purpose
  - Configuration comments explain WHY each setting
  - Makes onboarding faster
  - Reduces confusion when returning to file later


1️⃣3️⃣  RESTART POLICY
═══════════════════════════════════════════════════════════════════════════════

❌ WRONG:
   postgres:
     image: postgres:15-alpine
     # No restart policy, service dies and doesn't come back

✅ RIGHT (What we did):
   postgres:
     restart: unless-stopped

   kong-control-plane:
     restart: unless-stopped

   kong-gateway:
     restart: unless-stopped

WHY:
  - "unless-stopped" = auto-restart if container crashes
  - Except if explicitly stopped (podman-compose down)
  - Increases reliability and uptime
  - Handles temporary issues automatically
  - Options:
    * "no": Don't restart
    * "always": Always restart (even if explicitly stopped)
    * "unless-stopped": Restart unless stopped (best for most cases)
    * "on-failure": Restart only on failure with max retries


1️⃣4️⃣  DATABASE INITIALIZATION
═══════════════════════════════════════════════════════════════════════════════

❌ WRONG:
   postgres:
     image: postgres:15-alpine
     environment:
       POSTGRES_DB: kong
     # Default locale (C), no optimization

✅ RIGHT (What we did):
   postgres:
     environment:
       POSTGRES_INITDB_ARGS: >-
         --encoding=UTF8
         --locale=C
     # UTF8 for internationalization support
     # Locale C for consistent sorting and collation

WHY:
  - UTF8 encoding supports all languages and special characters
  - Locale C ensures consistent behavior across systems
  - Required for Kong to handle multi-language content
  - Prevents encoding errors with API responses


1️⃣5️⃣  IMAGE TAGS - STABILITY
═══════════════════════════════════════════════════════════════════════════════

❌ WRONG:
   postgres:
     image: postgres         # Latest, can break compatibility
   
   kong:
     image: kong/kong-gateway-enterprise  # No version, breaks between upgrades

✅ RIGHT (What we did):
   postgres:
     image: postgres:15-alpine    # Specific version 15, alpine (small)
   
   kong:
     image: kong/kong-gateway-enterprise:3.14  # Specific version

WHY:
  - Specific version prevents surprise breaking changes
  - "alpine" = smaller image, faster startup (80MB vs 300MB+)
  - Reproducible deployments across machines
  - Easier debugging (known versions)
  - Can control upgrade timing


1️⃣6️⃣  CONTAINER NAMES
═══════════════════════════════════════════════════════════════════════════════

❌ WRONG:
   postgres:
     # Auto-generated name like "project_db_1", confusing

✅ RIGHT (What we did):
   postgres:
     container_name: kong-postgres
   
   kong-control-plane:
     container_name: kong-control-plane
   
   kong-gateway:
     container_name: kong-gateway

WHY:
  - Explicit names are easier to remember
  - Easier to use podman exec / podman logs
  - Clearer in monitoring and debugging
  - Easy to find and reference


📋 SUMMARY TABLE - BEST PRACTICES CHECKLIST
═══════════════════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────┬─────────────┬──────────────────────┐
│ Practice                                │ Applied?    │ Benefit              │
├─────────────────────────────────────────┼─────────────┼──────────────────────┤
│ Compose version 3.8+                    │ ✓ Yes       │ All features enabled │
│ Health checks per service               │ ✓ Yes       │ Proper startup order  │
│ Service dependencies (depends_on)       │ ✓ Yes       │ No race conditions   │
│ Resource limits                         │ ✓ Yes       │ System stability     │
│ Resource reservations                   │ ✓ Yes       │ Guaranteed resources │
│ Structured logging (json-file driver)   │ ✓ Yes       │ No disk overflow     │
│ Environment variables with defaults     │ ✓ Yes       │ Flexibility          │
│ Separate internal/external networks     │ ✓ Yes       │ Security             │
│ Named volumes                           │ ✓ Yes       │ Persistent data      │
│ Correct service roles (CP vs DP)        │ ✓ Yes       │ Proper architecture  │
│ Port mapping with comments              │ ✓ Yes       │ Clarity              │
│ Specific image versions with tags       │ ✓ Yes       │ Reproducibility      │
│ Restart policy                          │ ✓ Yes       │ High availability    │
│ Explicit container names                │ ✓ Yes       │ Easy management      │
│ Security: No hardcoded secrets          │ ✓ Yes       │ Secure configs       │
│ Documentation and comments              │ ✓ Yes       │ Maintainability      │
└─────────────────────────────────────────┴─────────────┴──────────────────────┘


🎯 WHEN TO ADJUST THESE SETTINGS
═══════════════════════════════════════════════════════════════════════════════

For DEVELOPMENT (laptop/testing):
  - Current settings work fine
  - Reduce resource limits if you have < 4GB RAM

For STAGING (team testing):
  - Increase database resources: cpus: '2', memory: 2G
  - Enable HTTPS with certificates
  - Use realistic data volumes
  - Monitor logs daily

For PRODUCTION:
  - Increase all resource limits (depends on expected load)
  - Use external managed database (RDS, Cloud SQL)
  - Remove database from compose file
  - Use Kubernetes instead of Docker Compose
  - Implement proper CI/CD pipeline
  - Use secrets management (Vault, AWS Secrets Manager)
  - Add monitoring and alerting (Prometheus, Datadog)
  - Add backup strategy
  - Enable logging to centralized system (ELK, Splunk)


═══════════════════════════════════════════════════════════════════════════════
End of Best Practices Guide
═══════════════════════════════════════════════════════════════════════════════
