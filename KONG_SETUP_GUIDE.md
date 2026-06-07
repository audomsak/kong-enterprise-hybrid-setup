╔════════════════════════════════════════════════════════════════════════════════╗
║           Kong Enterprise 3.14 Hybrid Deployment - Setup Guide                ║
║                          Using Podman/Docker Compose                          ║
╚════════════════════════════════════════════════════════════════════════════════╝

📋 QUICK OVERVIEW
═══════════════════════════════════════════════════════════════════════════════

Your topology consists of 3 main components:

┌─────────────────────────────────────────────────────────────────┐
│                   Kong Control Plane (CP)                        │
│              - Admin API (port 8001)                             │
│              - Manager UI (port 8002)                            │
│              - Configuration management                          │
└─────────────────────────────────────────────────────────────────┘
                              ↑ ↓
                    Cluster Communication
                           (port 8005)
                              ↑ ↓
┌─────────────────────────────────────────────────────────────────┐
│               Kong Gateway (Data Plane - DP)                     │
│              - API Proxy (ports 8000, 8443)                      │
│              - Receives config from CP                           │
│              - Processes API requests only                       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                        Reads/Writes
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│              PostgreSQL Database (Shared)                        │
│              - Stores all Kong configurations                    │
│              - Accessible by both CP and DP                      │
└─────────────────────────────────────────────────────────────────┘


🔑 STEP 1: Get Your Kong Enterprise License
═══════════════════════════════════════════════════════════════════════════════

You MUST have a valid Kong Enterprise license to use this setup. The license is
a JSON object that contains your organization's entitlements.

Your license should look like this:
{
  "license": {
    "payload": {
      "admin_api": true,
      "billing": true,
      "rbac": true,
      ... (other features)
    },
    "license_expiration_date": 32503680000,
    "license_key": "your_actual_key_here"
  },
  "signature": "your_signature_here",
  "version": 3
}

💡 How to get the license:
   1. Contact Kong Sales (support@konghq.com)
   2. Request a trial license for Kong Gateway Enterprise
   3. You'll receive a license JSON file via email
   4. Keep this file safe - you'll use it to replace the placeholder


📝 STEP 2: Prepare Environment Variables
═══════════════════════════════════════════════════════════════════════════════

Create a file named `.env` in the same directory as the compose file:

------- .env (START) -------
# PostgreSQL password (change this to something secure!)
POSTGRES_PASSWORD=kong_secure_password_123

# Kong Enterprise License (paste your full license JSON here)
KONG_LICENSE_DATA='{"license":{"payload":{"admin_api":true,"billing":true,"bots":true,...}}}'
------- .env (END) -------

⚠️  IMPORTANT:
   - Keep the license key PRIVATE and SECURE
   - Don't share it in repositories or with others
   - Use strong passwords for PostgreSQL


🚀 STEP 3: Start the Services
═══════════════════════════════════════════════════════════════════════════════

1. Start all services:
   
   podman-compose -f kong-enterprise-hybrid-compose.yml up -d

2. Check if services are running:
   
   podman-compose -f kong-enterprise-hybrid-compose.yml ps

   You should see:
   - kong-postgres (running)
   - kong-control-plane (running)
   - kong-gateway (running)

3. Wait for all services to be healthy (first startup takes 30-60 seconds):
   
   podman logs -f kong-control-plane
   podman logs -f kong-gateway


✅ STEP 4: Verify Setup is Working
═══════════════════════════════════════════════════════════════════════════════

Test Control Plane Admin API:
   curl -s http://localhost:8001/status | jq .

   Expected response:
   {
     "database": {
       "reachable": true
     },
     "server": {
       "version": "3.14.0",
       "hostname": "kong-control-plane",
       "node_id": "your-node-id",
       "lua_version": "5.1",
       "configuration": {...}
     }
   }

Test Data Plane Gateway:
   curl -s http://localhost:8000/status | jq .

   Should return similar status


📊 STEP 5: Access Kong Manager (Admin UI)
═══════════════════════════════════════════════════════════════════════════════

Open your browser and go to:
   http://localhost:8002

Default credentials:
   Username: kong_admin
   Password: kong_admin (default, change in production!)

From here you can:
   ✓ Create services and routes
   ✓ Configure plugins
   ✓ Manage users and RBAC
   ✓ View analytics and logs


🔌 STEP 6: Understanding the Network Architecture
═══════════════════════════════════════════════════════════════════════════════

Two Networks are created:

1. kong_admin (172.20.0.0/16)
   - Private network
   - Used for Control Plane ↔ Data Plane communication
   - Used for Database access
   - NOT accessible from outside

2. kong_proxy (172.21.0.0/16)
   - Public network
   - Used for external clients to access Kong Gateway
   - Data Plane listens here for API requests


🔐 STEP 7: Configure HTTPS with Certificates
═══════════════════════════════════════════════════════════════════════════════

For production, you should enable TLS for cluster communication:

1. Generate certificates (run this on your host machine):
   
   docker run -v ./certs:/certs kong/kong-gateway:3.14 \
     kong generate-cert -d kong-control-plane

2. Uncomment the volume sections in the compose file:

   In kong-control-plane service:
   volumes:
     - ./certs/cluster-cert.pem:/etc/kong/cluster-cert.pem:ro
     - ./certs/cluster-cert-key.pem:/etc/kong/cluster-cert-key.pem:ro

   In kong-gateway service:
   volumes:
     - ./certs/cluster-cert.pem:/etc/kong/cluster-cert.pem:ro
     - ./certs/cluster-cert-key.pem:/etc/kong/cluster-cert-key.pem:ro

3. Restart services:
   
   podman-compose -f kong-enterprise-hybrid-compose.yml restart


📈 STEP 8: Scale to Multiple Data Planes (Optional)
═══════════════════════════════════════════════════════════════════════════════

To add another Kong Gateway instance:

1. Add this to your compose file:

   kong-gateway-2:
     image: kong/kong-gateway-enterprise:3.14
     container_name: kong-gateway-2
     # ... (same as kong-gateway, but use different ports)
     ports:
       - "8010:8000"  # Different HTTP port
       - "8010:8443"  # Different HTTPS port
     # ... rest of config

2. Both gateways will receive configs from the same Control Plane


🛠️  STEP 9: Common Commands for Managing the Setup
═══════════════════════════════════════════════════════════════════════════════

View logs:
   podman logs -f kong-control-plane    # See CP logs
   podman logs -f kong-gateway          # See DP logs
   podman logs -f kong-postgres         # See DB logs

Stop all services:
   podman-compose -f kong-enterprise-hybrid-compose.yml down

Remove everything (including data!):
   podman-compose -f kong-enterprise-hybrid-compose.yml down -v

Restart a single service:
   podman-compose -f kong-enterprise-hybrid-compose.yml restart kong-gateway

Access database directly:
   podman exec -it kong-postgres psql -U kong -d kong

Check cluster status:
   curl -s http://localhost:8001/clustering/status | jq .


⚠️  STEP 10: Important Notes & Best Practices
═══════════════════════════════════════════════════════════════════════════════

✓ DO:
  ✓ Use strong PostgreSQL passwords
  ✓ Keep your license key private and secure
  ✓ Enable HTTPS for production
  ✓ Use the resource limits defined in the compose file
  ✓ Monitor logs regularly
  ✓ Back up the PostgreSQL database regularly
  ✓ Change default admin credentials after first login

✗ DON'T:
  ✗ Expose port 8001 (Admin API) to public internet without authentication
  ✗ Use default passwords in production
  ✗ Commit license keys to version control
  ✗ Run without resource limits
  ✗ Share database credentials
  ✗ Use same certificates across multiple environments


📚 USEFUL REFERENCES
═══════════════════════════════════════════════════════════════════════════════

Kong Documentation: https://docs.konghq.com/gateway/
Kong Admin API Docs: https://docs.konghq.com/gateway/api/admin-ee/
Kong Manager UI Guide: https://docs.konghq.com/gateway/manager/
Podman Documentation: https://podman.io/
Docker Compose Docs: https://docs.docker.com/compose/


🆘 TROUBLESHOOTING
═══════════════════════════════════════════════════════════════════════════════

Problem: "License data is invalid"
Solution: Check that your KONG_LICENSE_DATA is a complete JSON object,
          properly formatted and not cut off.

Problem: "Can't connect to database"
Solution: Verify postgres service is healthy:
          podman logs kong-postgres
          Check POSTGRES_PASSWORD matches in .env and compose file

Problem: "Control Plane and Data Plane not communicating"
Solution: Check network connectivity:
          podman inspect kong-control-plane | grep -A 5 NetworkSettings

Problem: Gateway not receiving configuration updates
Solution: Check cluster status:
          curl -s http://localhost:8001/clustering/status | jq .
          Verify both services are in same network


═══════════════════════════════════════════════════════════════════════════════
Created for Kong Enterprise 3.14 with Podman
Last Updated: 2024
═══════════════════════════════════════════════════════════════════════════════
