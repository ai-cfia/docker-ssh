# Testing ALLOWED_DEST Functionality

This directory contains everything needed to test the new `ALLOWED_DEST` environment variable functionality for restricting SSH tunnel destinations. This guide combines setup instructions, testing procedures, and troubleshooting information into one comprehensive resource.

## Quick Start

### 🚀 One-Command Setup
```bash
./test-setup.sh
```

This will:
- Create a `keys/` directory with proper permissions (700)
- Generate an SSH key pair for testing in `test/keys/`
- Create a `.env.ssh` file with your public key
- Start all test services
- Run basic connectivity tests
- Run comprehensive tunnel tests
- Show testing examples

### 📋 What Gets Created

The test environment includes:

**4 SSH Servers** (different restriction levels):
- **Port 41222**: Unrestricted tunneling (`ALLOWED_DEST: "any"`)
- **Port 41223**: Restricted to nginx only (`ALLOWED_DEST: "nginx:80"`)
- **Port 41224**: No tunneling allowed (`ALLOWED_DEST: "none"`)
- **Port 41225**: Wildcard restrictions (`ALLOWED_DEST: "nginx:*"`)

**Test Service** (tunnel target):
- Nginx web server (port 80, also exposed on host port 41081 for direct access)

**Generated Files**:
- `keys/docker-ssh-test` - Private SSH key (permissions: 600)
- `keys/docker-ssh-test.pub` - Public SSH key (permissions: 644)
- `.env.ssh` - Environment file with AUTHORIZED_KEYS (permissions: 600)

## 🧪 Quick Tests

After running `./test-setup.sh`, try these commands:

```bash
# Should work - unrestricted server
ssh -i ./keys/docker-ssh-test -L 18080:nginx:80 -N tunnel@localhost -p 41222 &
curl http://localhost:18080

# Should work - restricted server, allowed destination
ssh -i ./keys/docker-ssh-test -L 18080:nginx:80 -N tunnel@localhost -p 41223 &
curl http://localhost:18080

# Should fail - restricted server, blocked destination
ssh -i ./keys/docker-ssh-test -L 18081:google.com:80 -N tunnel@localhost -p 41223 &
curl http://localhost:18081  # This should fail

# Clean up tunnels
pkill -f "ssh.*-L.*localhost"
```

## 🔧 Script Commands

The `test-setup.sh` script supports several commands:

```bash
./test-setup.sh setup     # Full setup (default)
./test-setup.sh start     # Start services only
./test-setup.sh test      # Run basic connectivity tests
./test-setup.sh test-all  # Run basic and comprehensive tunnel tests
./test-setup.sh examples  # Show testing examples
./test-setup.sh key-info  # Show SSH key information
./test-setup.sh cleanup   # Stop and clean up
./test-setup.sh help      # Show help
```

## 🔐 Security & Permissions

The setup script automatically sets proper permissions:

- **Keys directory**: `700` (only owner can read/write/execute)
- **Private key**: `600` (only owner can read/write)
- **Public key**: `644` (owner read/write, others read-only)
- **Environment file**: `600` (only owner can read/write)

## 🛠️ Manual Setup

If you prefer manual setup:

1. **Generate SSH keys**:
   ```bash
   mkdir -p keys
   chmod 700 keys
   ssh-keygen -t ed25519 -f keys/docker-ssh-test -N ""
   chmod 600 keys/docker-ssh-test
   chmod 644 keys/docker-ssh-test.pub
   ```

2. **Create environment file**:
   ```bash
   echo "AUTHORIZED_KEYS=$(cat keys/docker-ssh-test.pub)" > .env.ssh
   chmod 600 .env.ssh
   ```

3. **Start services**:
   ```bash
   docker compose -f docker-compose.test.yml up -d
   ```

## Comprehensive Test Scenarios

The test environment includes 4 different SSH server configurations and 1 simple web service:

### Scenario 1: Unrestricted Tunneling (Port 41222)
**Configuration:** `ALLOWED_DEST: "any"`
**Expected:** All tunnel connections should work

```bash
# Test web tunneling (should work)
ssh -i ./keys/docker-ssh-test -L 18080:nginx:80 -N tunnel@localhost -p 41222 &
curl http://localhost:18080

# Test with different port (should work)
ssh -i ./keys/docker-ssh-test -L 18081:nginx:80 -N tunnel@localhost -p 41222 &
curl http://localhost:18081

# Clean up tunnels
pkill -f "ssh.*41222"
```

### Scenario 2: Restricted Tunneling (Port 41223)
**Configuration:** `ALLOWED_DEST: "nginx:80"`
**Expected:** Only nginx:80 should be accessible

```bash
# Test allowed service - nginx:80 (should work)
ssh -i ./keys/docker-ssh-test -L 18080:nginx:80 -N tunnel@localhost -p 41223 &
curl http://localhost:18080

# Test blocked destination - different port (should fail)
ssh -i ./keys/docker-ssh-test -L 18081:nginx:8080 -N tunnel@localhost -p 41223 &
curl http://localhost:18081  # This should fail

# Test blocked destination - different host (should fail) 
ssh -i ./keys/docker-ssh-test -L 18082:google.com:80 -N tunnel@localhost -p 41223 &
curl http://localhost:18082  # This should fail

# Clean up tunnels
pkill -f "ssh.*41223"
```

### Scenario 3: No Tunneling Allowed (Port 41224)
**Configuration:** `ALLOWED_DEST: "none"`
**Expected:** All tunnel attempts should be denied

```bash
# All these should fail
ssh -i ./keys/docker-ssh-test -L 18080:nginx:80 -N tunnel@localhost -p 41224 &
ssh -i ./keys/docker-ssh-test -L 18081:google.com:80 -N tunnel@localhost -p 41224 &

# Verify SSH connection still works (just no tunneling)
ssh -i ./keys/docker-ssh-test tunnel@localhost -p 41224 "echo 'SSH works but no tunneling'"

# Clean up
pkill -f "ssh.*41224"
```

### Scenario 4: Wildcard Port Restrictions (Port 41225)
**Configuration:** `ALLOWED_DEST: "nginx:*"`
**Expected:** Only nginx host should be accessible (any port)

```bash
# Test allowed host with standard port (should work)
ssh -i ./keys/docker-ssh-test -L 18080:nginx:80 -N tunnel@localhost -p 41225 &
curl http://localhost:18080

# Test allowed host with different port (should work)
ssh -i ./keys/docker-ssh-test -L 18081:nginx:8080 -N tunnel@localhost -p 41225 &
# This tunnel should establish even though nginx isn't actually listening on 8080

# Test blocked host (should fail)
ssh -i ./keys/docker-ssh-test -L 18082:google.com:80 -N tunnel@localhost -p 41225 &
curl http://localhost:18082  # Should fail

# Clean up
pkill -f "ssh.*41225"
```

## Testing with Different Tools

### Web Testing
```bash
# HTTP tunnel test
curl http://localhost:18080
curl http://localhost:18080/api/health

# Using browser
open http://localhost:18080  # macOS
xdg-open http://localhost:18080  # Linux

# Test with wget
wget -O - http://localhost:18080

# Test with browser developer tools
# Navigate to http://localhost:18080 and check network tab
```

### Advanced Web Testing
```bash
# Test HTTP methods
curl -X POST http://localhost:18080/api/health
curl -X GET http://localhost:18080/api/health

# Test with headers
curl -H "User-Agent: Test-Agent" http://localhost:18080

# Download content through tunnel
curl -o test-page.html http://localhost:18080
```

## Expected Behaviors

### ✅ Should Work
- SSH connections to all servers (authentication and shell access)
- Tunneling to allowed destinations based on `ALLOWED_DEST` configuration
- Multiple simultaneous tunnels to allowed destinations
- SSH agent forwarding (if enabled)

### ❌ Should Fail
- Tunneling to destinations not listed in `ALLOWED_DEST`
- Port forwarding when `ALLOWED_DEST: "none"`
- Bypassing restrictions through different ports (unless wildcards used)
- Tunneling to external hosts when only specific internal hosts are allowed

## 🐛 Troubleshooting

### SSH Key Issues
```bash
# Check key permissions
./test-setup.sh key-info

# Regenerate SSH keys
rm -rf keys/ .env.ssh
./test-setup.sh setup
```

### Service Issues
```bash
# Check service status
docker compose -f docker-compose.test.yml ps

# View logs for specific SSH server
docker compose -f docker-compose.test.yml logs ssh-restricted

# Follow logs in real-time
docker compose -f docker-compose.test.yml logs -f ssh-restricted

# Restart services
docker compose -f docker-compose.test.yml restart
```

### Verify SSH Configuration
```bash
# Connect and check SSH configuration
ssh -i ./keys/docker-ssh-test tunnel@localhost -p 41223 "cat /etc/ssh/sshd_config.d/custom.conf"

# Test basic SSH connection (should always work)
ssh -i ./keys/docker-ssh-test tunnel@localhost -p 41223 "echo 'Connection successful'"
```

### Test Direct Access (Baseline)
```bash
# Test direct access to nginx (should work)
curl http://localhost:41081

# Compare with tunneled access
ssh -i ./keys/docker-ssh-test -L 18080:nginx:80 -N tunnel@localhost -p 41222 &
curl http://localhost:18080
```

### Permission Issues
The script automatically sets correct permissions, but if you encounter issues:
```bash
chmod 700 keys/
chmod 600 keys/docker-ssh-test .env.ssh
chmod 644 keys/docker-ssh-test.pub
```

### Common Issues

1. **Permission denied (publickey)**
   - Ensure your public key is correctly added to `AUTHORIZED_KEYS`
   - Check the SSH key file permissions

2. **Connection refused**
   - Verify the container is running: `docker compose -f docker-compose.test.yml ps`
   - Check if the port is exposed correctly

3. **Tunnel established but can't connect to service**
   - This is expected behavior when testing restricted destinations
   - Check the SSH server logs to confirm the restriction is working

4. **"channel 2: open failed: administratively prohibited"**
   - This error indicates the ALLOWED_DEST restriction is working correctly
   - The tunnel destination is blocked by SSH configuration

## Advanced Testing

### Testing with Different Destination Formats
```bash
# Test with IP addresses (if you know the container IP)
docker inspect test-nginx | grep IPAddress
ssh -i ./keys/docker-ssh-test -L 18080:172.20.0.2:80 -N tunnel@localhost -p 41223

# Test with localhost (should be blocked in restricted mode)
ssh -i ./keys/docker-ssh-test -L 18080:localhost:80 -N tunnel@localhost -p 41223
```

### Testing Invalid Configurations
Test how the system handles invalid `ALLOWED_DEST` values by modifying the docker-compose file:

```yaml
# Invalid formats that should cause startup errors
ALLOWED_DEST: "invalid-format"
ALLOWED_DEST: "host:invalid-port"
ALLOWED_DEST: "host:"
```

### Performance Testing
```bash
# Test multiple simultaneous connections
for i in {1..5}; do
  ssh -i ./keys/docker-ssh-test -L $((18080+i)):nginx:80 -N tunnel@localhost -p 41222 &
done

# Test all tunnels work
for i in {1..5}; do
  curl http://localhost:$((18080+i)) &
done
```

### Testing Edge Cases
```bash
# Test very long hostnames (should be handled gracefully)
ssh -i ./keys/docker-ssh-test -L 18080:this-is-a-very-long-hostname-that-should-be-rejected:80 -N tunnel@localhost -p 41223

# Test special characters in hostnames
ssh -i ./keys/docker-ssh-test -L 18080:nginx_test:80 -N tunnel@localhost -p 41223

# Test high port numbers
ssh -i ./keys/docker-ssh-test -L 18080:nginx:65535 -N tunnel@localhost -p 41225
```

## Clean Up
```bash
# Stop all services
docker compose -f docker-compose.test.yml down

# Remove volumes (optional - removes persistent data)
docker compose -f docker-compose.test.yml down -v

# Kill any remaining SSH tunnels
pkill -f "ssh.*-L.*localhost"

# Or use the script
./test-setup.sh cleanup
```

## 📁 File Structure

```
test/
├── test-setup.sh           # Main setup script
├── docker-compose.test.yml # Docker Compose configuration
├── test-nginx.conf         # Nginx configuration
├── .gitignore             # Excludes keys and env files
├── COMBINED-README.md     # This comprehensive guide
├── keys/                  # SSH keys (auto-generated)
│   ├── docker-ssh-test    # Private key (600)
│   └── docker-ssh-test.pub # Public key (644)
└── .env.ssh              # Environment file (600, auto-generated)
```

## 🎯 Expected Results

| Scenario | Port | nginx:80 | google.com:80 | nginx:8080 |
|----------|------|----------|---------------|------------|
| Unrestricted | 41222 | ✅ | ✅ | ✅ |
| Restricted | 41223 | ✅ | ❌ | ❌ |
| No Tunnels | 41224 | ❌ | ❌ | ❌ |
| Wildcard | 41225 | ✅ | ❌ | ✅ |

Happy testing! 🚀 