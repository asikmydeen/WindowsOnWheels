# WindowsOnWheels

# Complete Guide: Remote Windows Access via Raspberry Pi with Tailscale

This guide enables you to access your Windows PC from anywhere using RDP, even from restricted networks or VPNs, by routing through a Raspberry Pi at home using Tailscale.

## 🎯 What This Setup Achieves

- Access your Windows PC from any device with RDP client (no Tailscale needed on client)
- Works through corporate firewalls and VPNs
- Secure encrypted connection via Tailscale
- Automatic failover and monitoring
- No monthly fees or subscriptions

## 📋 Prerequisites

### Hardware
- **Raspberry Pi**: Pi 4 (4GB+ RAM) or Pi 5
- **Storage**: 32GB+ A1-rated microSD card or USB 3.0 SSD
- **Power Supply**: Official Raspberry Pi power supply
- **Windows PC**: Windows 11 Pro (Home edition doesn't support RDP server)
- **Network**: Stable home internet with router admin access

### Accounts
- Tailscale account (free tier is sufficient)
- Router admin credentials
- Windows admin account with password

---

## Part 1: Raspberry Pi Setup

### Step 1: Install Raspberry Pi OS

1. Download Raspberry Pi Imager from https://www.raspberrypi.com/software/
2. Flash **Raspberry Pi OS Lite (64-bit)** to your SD card
3. Before ejecting, enable SSH:
   - Click gear icon in Imager
   - Enable SSH with password authentication
   - Set hostname: `pi-tailscale`
   - Configure your WiFi if not using Ethernet

### Step 2: Initial Pi Configuration

SSH into your Pi:
```bash
ssh pi@pi-tailscale.local
# Or use the IP address from your router
```

Run initial setup:
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install essential packages
sudo apt install -y curl wget htop iotop vnstat fail2ban iptables-persistent git bc

# Configure automatic security updates
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

### Step 3: Enable IP Forwarding (Critical!)

```bash
# Enable IP forwarding - REQUIRED for exit node functionality
cat << EOF | sudo tee /etc/sysctl.d/99-tailscale.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
net.core.rmem_max = 26214400
net.core.wmem_max = 26214400
net.ipv4.tcp_congestion_control = bbr
EOF

# Apply settings
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

# Verify IP forwarding is enabled
sysctl net.ipv4.ip_forward
# Should output: net.ipv4.ip_forward = 1
```

### Step 4: Install and Configure Tailscale

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Start Tailscale with exit node enabled
sudo tailscale up --advertise-exit-node --advertise-routes=192.168.0.0/16,10.0.0.0/8 --ssh

# This will display a login URL - open it in your browser to authenticate
```

### Step 5: Approve Exit Node in Admin Console

1. Go to https://login.tailscale.com/admin/machines
2. Find your Pi device
3. Click the three dots menu → "Edit route settings"
4. Enable:
   - ✅ Use as exit node
   - ✅ Advertised subnets

### Step 6: Note Your Pi's Network Details

```bash
# Get your Pi's local IP (you'll need this for router setup)
hostname -I | awk '{print $1}'
# Example output: 10.0.0.84

# Get your home's public IP (for RDP connection)
curl ifconfig.me
# Example output: 98.232.34.39

# Get your Pi's Tailscale name
tailscale status | grep pi-tailscale
```

---

## Part 2: Windows PC Setup

### Step 1: Enable Remote Desktop

Run PowerShell as Administrator:

```powershell
# Enable Remote Desktop
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0

# Configure RDP security
$rdpPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp"
Set-ItemProperty -Path $rdpPath -Name "UserAuthentication" -Value 1  # Require NLA
Set-ItemProperty -Path $rdpPath -Name "SecurityLayer" -Value 2      # Require TLS

# Start RDP service
Start-Service TermService

# Ensure your Windows account has a password
# If you use Windows Hello PIN only, you need to set a password:
# net user $env:USERNAME *
```

### Step 2: Install Tailscale on Windows

1. Download from https://tailscale.com/download/windows
2. Install and sign in with the **same account** used on the Pi
3. Verify connection: `tailscale status`

### Step 3: Configure Windows Firewall

```powershell
# Create Tailscale-only RDP rule
New-NetFirewallRule -DisplayName "RDP-Tailscale-Only" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 3389 `
    -RemoteAddress 100.64.0.0/10 `
    -Action Allow `
    -Profile Any

# Allow Tailscale direct connections
New-NetFirewallRule -DisplayName "Tailscale Direct" `
    -Direction Inbound `
    -Protocol UDP `
    -LocalPort 41641 `
    -Action Allow `
    -Profile Any
```

### Step 4: Create Monitoring Script

Create folder `C:\TailscaleScripts` and save this as `TailscaleMonitor.ps1`:

```powershell
# Simple Tailscale connection monitor
$logPath = "C:\TailscaleScripts\tailscale.log"

function Write-Log {
    param($Message)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    "$timestamp - $Message" | Add-Content $logPath
}

# Check Tailscale connection
$status = & tailscale status 2>&1
if ($LASTEXITCODE -ne 0) {
    Write-Log "Tailscale not connected, attempting to reconnect"
    & tailscale up
}

Write-Log "Tailscale status check completed"
```

### Step 5: Schedule Monitoring Task

```powershell
# Create scheduled task for monitoring
$action = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument "-WindowStyle Hidden -ExecutionPolicy Bypass -File C:\TailscaleScripts\TailscaleMonitor.ps1"

$trigger = New-ScheduledTaskTrigger -Daily -At 9am -RepetitionInterval (New-TimeSpan -Minutes 30)
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -RunLevel Highest

Register-ScheduledTask -TaskName "TailscaleMonitor" `
    -Action $action -Trigger $trigger -Principal $principal
```

### Step 6: Note Your Windows PC's Tailscale IP

```powershell
# Get Windows PC's Tailscale IP
tailscale ip -4
# Example output: 100.78.136.73
```

---

## Part 3: Configure Port Forwarding

### Step 1: Set Up Pi Port Forwarding

On your Raspberry Pi:

```bash
# Forward incoming RDP connections to Windows PC
sudo iptables -t nat -A PREROUTING -p tcp --dport 3389 -j DNAT --to-destination 100.78.136.73:3389
sudo iptables -A FORWARD -p tcp -d 100.78.136.73 --dport 3389 -j ACCEPT

# Enable masquerading (CRITICAL - this makes return traffic work)
sudo iptables -t nat -A POSTROUTING -d 100.78.136.73 -p tcp --dport 3389 -j MASQUERADE

# Save the rules permanently
sudo netfilter-persistent save

# Verify rules are in place
sudo iptables -t nat -L -n -v
```

Replace `100.78.136.73` with your Windows PC's actual Tailscale IP.

### Step 2: Configure Router Port Forwarding

1. Log into your router's admin panel (usually 192.168.1.1 or 10.0.0.1)
2. Find "Port Forwarding" or "Virtual Server" section
3. Create new rule:
   - **Service Name**: Windows RDP
   - **External Port**: 3389
   - **Internal IP**: Your Pi's local IP (e.g., 10.0.0.84)
   - **Internal Port**: 3389
   - **Protocol**: TCP
4. Save and apply

---

## Part 4: Security Hardening

### On Raspberry Pi:

```bash
# Install and configure fail2ban for brute force protection
sudo apt install fail2ban -y

# Create RDP protection
sudo tee /etc/fail2ban/jail.d/rdp-forward.conf << 'EOF'
[rdp-forward]
enabled = true
port = 3389
filter = rdp-forward
logpath = /var/log/syslog
maxretry = 5
bantime = 3600
findtime = 600
EOF

sudo tee /etc/fail2ban/filter.d/rdp-forward.conf << 'EOF'
[Definition]
failregex = .*DNAT.*DPT=3389.*SRC=<HOST>
ignoreregex =
EOF

sudo systemctl restart fail2ban

# Create monitoring script
cat << 'EOF' | sudo tee /usr/local/bin/rdp-monitor.sh
#!/bin/bash
LOG="/var/log/rdp-monitor.log"

# Check Windows PC connectivity
if ! tailscale ping -c 1 100.78.136.73 > /dev/null 2>&1; then
    echo "$(date): WARNING - Cannot reach Windows PC" >> $LOG
fi

# Check RDP port
if ! nc -zv 100.78.136.73 3389 > /dev/null 2>&1; then
    echo "$(date): WARNING - RDP port not responding" >> $LOG
fi

# Log connection count
CONNECTIONS=$(sudo iptables -t nat -L PREROUTING -n -v | grep "dpt:3389" | awk '{print $1}')
echo "$(date): Total RDP connections: $CONNECTIONS" >> $LOG
EOF

sudo chmod +x /usr/local/bin/rdp-monitor.sh

# Schedule monitoring
(crontab -l 2>/dev/null; echo "*/15 * * * * /usr/local/bin/rdp-monitor.sh") | crontab -
```

---

## Part 5: Testing Your Setup

### Step 1: Verify Everything is Working

On your Raspberry Pi:
```bash
# Check Tailscale status
tailscale status

# Test Windows PC connectivity
tailscale ping [your-windows-tailscale-ip]

# Verify port forwarding rules
sudo iptables -t nat -L PREROUTING -n -v

# Test RDP port
nc -zv [your-windows-tailscale-ip] 3389
```

### Step 2: Connect from External Device

From any device with RDP client (no Tailscale needed):

1. Open your RDP client
2. Connect to: `[your-home-public-ip]:3389`
3. Username: Your Windows username
4. Password: Your Windows password

Example:
- Server: `98.232.34.39`
- Username: `Asik`
- Password: `YourPassword`

---

## Part 6: Maintenance and Monitoring

### Create Status Check Script on Pi

```bash
cat << 'EOF' > ~/check-rdp-status.sh
#!/bin/bash
echo "=== RDP Forwarding Status ==="
echo "Public IP: $(curl -s ifconfig.me)"
echo "Pi Local IP: $(hostname -I | awk '{print $1}')"
echo ""
echo "=== Port Forwarding Rules ==="
sudo iptables -t nat -L PREROUTING -n -v | grep 3389
echo ""
echo "=== Tailscale Status ==="
tailscale status | grep -E "(windows|desktop)"
echo ""
echo "=== Connection Test ==="
nc -zv 100.78.136.73 3389 2>&1
echo ""
echo "=== Recent Connections ==="
tail -n 10 /var/log/rdp-monitor.log 2>/dev/null
echo ""
echo "=== Blocked IPs ==="
sudo fail2ban-client status rdp-forward 2>/dev/null | grep "Banned IP"
EOF

chmod +x ~/check-rdp-status.sh
```

### Regular Maintenance Tasks

**Weekly:**
- Run `~/check-rdp-status.sh` to verify status
- Check for banned IPs: `sudo fail2ban-client status rdp-forward`
- Review logs: `tail -n 50 /var/log/rdp-monitor.log`

**Monthly:**
- Update system: `sudo apt update && sudo apt upgrade`
- Check Tailscale updates: `tailscale version`
- Verify firewall rules survived reboots

---

## Troubleshooting

### Connection Issues

1. **Can't connect to RDP:**
   ```bash
   # On Pi, check if packets are arriving
   sudo tcpdump -i any -n port 3389
   ```

2. **Verify masquerading is active:**
   ```bash
   sudo iptables -t nat -L POSTROUTING -n -v
   # Should show MASQUERADE rule
   ```

3. **Test from Pi directly:**
   ```bash
   telnet [windows-tailscale-ip] 3389
   ```

### Common Fixes

- **Windows PC offline:** Check Tailscale status on Windows
- **Port forwarding not working:** Verify router settings and Pi local IP hasn't changed
- **RDP authentication fails:** Ensure Windows password is set (not just PIN)

---

## Optional Enhancements

### Dynamic DNS (if your home IP changes)

```bash
# Install DuckDNS updater
mkdir ~/duckdns
cd ~/duckdns
echo "curl -s \"https://www.duckdns.org/update?domains=YOURDOMAIN&token=YOURTOKEN&ip=\" -o ~/duckdns/duck.log" > duck.sh
chmod 700 duck.sh
(crontab -l 2>/dev/null; echo "*/5 * * * * ~/duckdns/duck.sh") | crontab -
```

Then connect using: `yourdomain.duckdns.org` instead of IP

### Alternative Port (for extra security)

Use port 13389 instead of 3389:
1. Update Pi iptables rules to use 13389
2. Change router forwarding to 13389
3. Connect using: `your-ip:13389`

---

## Security Best Practices

1. **Use strong passwords** on Windows account
2. **Monitor logs** regularly for unauthorized attempts
3. **Keep everything updated** - OS, Tailscale, and firmware
4. **Consider VPN** as additional layer if handling sensitive data
5. **Backup your configuration** - especially iptables rules

---

## Summary

You now have a secure, reliable way to access your Windows PC from anywhere, even through restrictive networks and VPNs. The setup routes your RDP connection through your home network via Tailscale's secure mesh network, providing both security and accessibility.

**Key Points:**
- Works from any device with RDP client
- No Tailscale needed on client device
- Encrypted via Tailscale between Pi and Windows
- Monitored and protected against brute force
- Works through firewalls and VPNs
