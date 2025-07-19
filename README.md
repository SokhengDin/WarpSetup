# Complete Cloudflare Zero Trust & WARP Connector Setup Guide

## Overview
Successfully configured a complete Zero Trust setup with:
- **Domain**: redalab.org 
- **Cloudflared Tunnel**: For public access to services
- **WARP Connector**: For private VPN access to rack server network
- **Rack Server Network**: 192.168.10.0/24 (Server IP: 192.168.10.46)

## Architecture
```
Internet Users → Cloudflare Tunnel → app.redalab.org (Public Access)
Mac with WARP → Cloudflare Edge → WARP Connector → 192.168.10.46 (Private Access)
```

---

## Part 1: Initial Setup

### 1. Domain Configuration
- **Purchased**: redalab.org
- **Added to Cloudflare**: Changed nameservers to Cloudflare
- **DNS Management**: Through Cloudflare dashboard

### 2. Zero Trust Organization Setup
- **Created Zero Trust team**: "redalab"
- **Team domain**: redalab.cloudflareaccess.com
- **Plan**: Free tier (50 users)

---

## Part 2: Cloudflared Tunnel Setup (Public Access)

### Docker Compose Configuration
```yaml
version: '3.8'

services:
  cloudflare-tunnel:
    image: cloudflare/cloudflared:latest
    container_name: cloudflare-tunnel
    restart: unless-stopped
    command: tunnel --no-autoupdate run --token ${TUNNEL_TOKEN}
    network_mode: host
    environment:
      - TUNNEL_TOKEN=${TUNNEL_TOKEN}
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    healthcheck:
      test: ["CMD-SHELL", "cloudflared tunnel info --token $TUNNEL_TOKEN || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

# Note: Using host networking removes some security isolation
# but resolves conflicts between WARP client and cloudflared
```

### Environment File (.env)
```bash
TUNNEL_TOKEN=your_actual_tunnel_token_here
```

### Auto-start Service
```bash
# Create systemd service for auto-start
sudo nano /etc/systemd/system/cloudflare-tunnel.service

# Enable auto-start
sudo systemctl enable cloudflare-tunnel.service
sudo systemctl start cloudflare-tunnel.service
```

---

## Part 3: WARP Client Installation (Rack Server)

### Install WARP Client
```bash
# Add Cloudflare repository
curl -fsSL https://pkg.cloudflareclient.com/pubkey.gpg | sudo gpg --yes --dearmor --output /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] https://pkg.cloudflareclient.com/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflare-client.list

# Update and install
sudo apt update
sudo apt install cloudflare-warp

# Fix dependencies if needed
sudo apt install gnupg2 libnss3-tools && sudo apt --fix-broken install
```

---

## Part 4: WARP Connector Setup

### Zero Trust Dashboard Configuration

#### Step 1: Enable Prerequisites
- **Settings → Network → Firewall**
- **Enable "Proxy"** ✅ (CRITICAL - without this, routing fails)
- **Check**: TCP, UDP, ICMP

#### Step 2: Enable Global Settings
- **Allow WARP to WARP connection**: ON
- **Override local interface IP**: ON

#### Step 3: Create WARP Connector Tunnel
- **Networks → Tunnels → Create tunnel**
- **Type**: WARP Connector
- **Name**: "rack-server-connector"

### Server Configuration

#### Connect WARP Connector
```bash
# Register as WARP Connector (using token from dashboard)
warp-cli connector new YOUR_CONNECTOR_TOKEN

# Connect
warp-cli connect

# Verify status
warp-cli status
# Should show: "Connected" with team "redalab"
```

#### Enable IP Forwarding
```bash
# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1

# Make permanent
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
```

#### Fix iptables (CRITICAL)
```bash
# Check current FORWARD policy
sudo iptables -L FORWARD

# Set FORWARD policy to ACCEPT (was DROP - this was the main issue)
sudo iptables -P FORWARD ACCEPT

# Add specific WARP interface rules
sudo iptables -I FORWARD -i CloudflareWARP -j ACCEPT
sudo iptables -I FORWARD -o CloudflareWARP -j ACCEPT

# Save rules
sudo mkdir -p /etc/iptables
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

---

## Part 5: Network Routes Configuration

### Zero Trust Dashboard Routes
- **Networks → Routes → Create route**
- **CIDR**: `192.168.10.0/24`
- **Tunnel**: Select WARP Connector tunnel
- **Description**: "Rack Server Network"

### Split Tunnels Configuration
- **Settings → WARP Client → Device settings → Default profile**
- **Split Tunnels → Include IPs and domains**
- **Add**:
  - `192.168.10.0/24` (Rack Server Network)
  - `100.96.0.0/12` (WARP to WARP Communication)

---

## Part 6: User Device Setup (Mac)

### Install WARP Client
1. Download from https://1.1.1.1/
2. Install Cloudflare WARP app

### Connect to Zero Trust
1. **WARP Settings → Account → Login with Cloudflare Zero Trust**
2. **Team name**: `redalab`
3. **Authenticate** with your credentials
4. **Connect WARP**

---

## Final Working Configuration

### Server Status
```bash
# Check WARP Connector status
warp-cli status
# Output: Status update: Connected

# Check registration
warp-cli registration show
# Shows: Organization: redalab, Auth Method: Warp Connector Token

# Check interface
ip addr show CloudflareWARP
# Shows: inet 100.96.0.2/32 scope global CloudflareWARP
```

### Network Test Results
```bash
# From Mac - WARP to WARP communication
ping 100.96.0.2
# ✅ SUCCESS

# From Mac - Access rack server
ping 192.168.10.46
# ✅ SUCCESS

# SSH access
ssh user@192.168.10.46
# ✅ SUCCESS

# Web services access
curl http://192.168.10.46:3000
# ✅ SUCCESS
```

---

## Key Issues Resolved

### 1. Docker Permission Issues
```bash
# Fix Docker permissions
sudo usermod -aG docker $USER && \
sudo systemctl start docker && \
sudo systemctl enable docker && \
sudo chmod 666 /var/run/docker.sock && \
newgrp docker
```

### 2. WARP Client Conflicts
- **Solution**: Used host networking in Docker Compose
- **Separated concerns**: Cloudflared tunnel for public, WARP Connector for private

### 3. Split Tunnel Configuration
- **Issue**: Missing `100.96.0.0/12` range in Include mode
- **Solution**: Added both rack network and WARP-to-WARP ranges

### 4. iptables FORWARD Policy (Main Issue)
- **Issue**: FORWARD policy was DROP, blocking all traffic
- **Solution**: Set to ACCEPT and added WARP interface rules

### 5. Network Proxy Setting (Critical)
- **Issue**: Proxy was disabled in Zero Trust Network settings
- **Solution**: Enabled Proxy with TCP, UDP, ICMP

---

## Benefits Achieved

✅ **Secure Remote Access**: VPN-like access to rack server from anywhere  
✅ **Public Service Access**: Apps accessible via redalab.org  
✅ **Zero Port Forwarding**: No router configuration needed  
✅ **Encrypted Traffic**: All traffic encrypted through Cloudflare  
✅ **Hidden Origin IP**: Real server IP addresses are protected  
✅ **Free Tier**: Up to 50 users at no cost  
✅ **Auto-Recovery**: Services restart automatically on boot  

---

## Maintenance Commands

### Check Service Status
```bash
# Tunnel status
docker-compose ps
docker-compose logs cloudflare-tunnel

# WARP Connector status
warp-cli status
sudo journalctl -u warp-svc -n 20
```

### Restart Services
```bash
# Restart tunnel
docker-compose restart cloudflare-tunnel

# Restart WARP
warp-cli disconnect && warp-cli connect
```

### View Logs
```bash
# Tunnel logs
docker-compose logs -f cloudflare-tunnel

# WARP logs
sudo journalctl -u warp-svc -f
```

---

## Final Architecture Summary

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Internet      │    │   Cloudflare    │    │   Rack Server   │
│   Users         │───▶│   Edge/Tunnel   │───▶│   Services      │
│                 │    │                 │    │   redalab.org   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                        │
┌─────────────────┐    ┌─────────────────┐             │
│   Mac with      │    │   Cloudflare    │             │
│   WARP Client   │───▶│   Edge/WARP     │─────────────┘
│   100.96.0.4    │    │   Connector     │   192.168.10.46
└─────────────────┘    └─────────────────┘   100.96.0.2
```

**Result**: Complete Zero Trust network with both public and private access capabilities!
