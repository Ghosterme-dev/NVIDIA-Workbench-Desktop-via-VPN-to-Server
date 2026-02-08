# NVIDIA-Workbench-Desktop-via-VPN-to-Server
This guide provides a working solution for connecting the NVIDIA AI Workbench Desktop App to a remote server through an SSH tunnel over a VPN connection

Unfortunately, a pure VPN tunnel via WireGuard does not work with Workbench due to fixed and unchangeable timeouts (hardcoded). Only SSH works when sent through the VPN tunnel on WireGuard.
My requirement was also that I wanted to connect to the Spark server using Workbench Desktop (because I am familiar with the GUI). To do this, however, you have to trick Workbench and configure it in the background (not via the GUI). Incidentally, since Workbench unfortunately has fixed ports, you also have to redirect them, etc.

If you find any discrepancies, please let me know and I will correct the instructions.

Here is the solution that works for me (macOS): 

## Complete Step-by-Step Configuration Guide

This guide provides a working solution for connecting the NVIDIA AI Workbench Desktop App to a remote server through an SSH tunnel over a VPN connection.[^1][^2]

***

## Use Case

This configuration is ideal when:

- Direct SSH access to the remote server times out or is unstable
- VPN connection is available but has keepalive issues
- You need to access a remote GPU server through corporate VPN
- VSCode Remote-SSH works but NVIDIA Workbench direct connection fails

***

## Prerequisites

✅ VPN client installed and configured (e.g., WireGuard, OpenVPN)
✅ VPN connection stable enough to maintain basic connectivity
✅ SSH key-based authentication configured (password auth not supported by Workbench)[^1]
✅ VSCode with Remote-SSH Extension (recommended for verification)
✅ NVIDIA AI Workbench Desktop App installed on local machine
✅ NVIDIA AI Workbench installed on remote server

***

## Architecture Overview 
_Unfortunately, this diagram is not ideal, as the arrows leading away from localhost should of course also go through the tunnel_

```
Local Machine                SSH Tunnel              Remote Server
┌─────────────┐             ┌──────────┐           ┌──────────────┐
│             │             │   VPN    │           │              │
│ Workbench   │────────────▶│  Tunnel  │──────────▶│  wb-svc      │
│ Desktop App │             │          │           │  (port 10001)│
│             │             └──────────┘           │              │
│ localhost:  │                                    │ Remote IP:   │
│  - 10005 ───┼────────────────────────────────────▶ 10001        │
│  - 10004 ───┼────────────────────────────────────▶ 8888         │
│  - 10022 ───┼────────────────────────────────────▶ 22           │
└─────────────┘                                    └──────────────┘
```

**Port Mapping**:

- Local 10005 → Remote 10001 (wb-svc Service API)[^1]
- Local 10004 → Remote 8888 (Proxy/Jupyter for applications)[^2]
- Local 10022 → Remote 22 (SSH for fingerprint verification)

***

## Step 1: Connect and Verify VPN

```bash
# Connect VPN (method depends on your VPN client)
# Example for WireGuard: Start via GUI or CLI

# Verify connectivity to remote server
ping -c 3 <REMOTE_SERVER_IP>
```

**Expected output**: Successful ping responses

⚠️ **Critical**: Without an active VPN connection, all subsequent steps will fail.

***

## Step 2: Connect VSCode to Remote Server (Verification)

This step verifies that SSH connectivity works through the VPN.[^3]

```bash
# Open VSCode
# Press Cmd+Shift+P (Mac) or Ctrl+Shift+P (Windows/Linux)
# Type: "Remote-SSH: Connect to Host"
# Select your remote server from the list
```

Wait until VSCode establishes a stable connection to the remote server.

***

## Step 3: Start wb-svc on Remote Server

**In VSCode Terminal** (connected to remote server):

```bash
# Check if wb-svc is already running
ps aux | grep wb-svc

# If not running, start it:
~/.nvwb/bin/wb-svc serve &

# Verify it's running (should show port 10001)
ps aux | grep wb-svc
```

**Expected output**:

```
username    2614  ... wb-svc -quiet -port 10001 serve
```

**Test locally on remote server**:

```bash
curl http://localhost:10001/v1/version
```

**Expected response**:

```json
{"built_on":"Thu Oct 30 17:38:11 UTC 2025","channel":"stable","version":"0.80.4"}
```


***

## Step 4: Create SSH Tunnel from Local Machine

**On your local machine** (Terminal/Command Prompt):

```bash
# Complete SSH tunnel with 3 port forwards
ssh -i "/path/to/your/ssh_private_key" \
    -o ServerAliveInterval=15 \
    -o ServerAliveCountMax=4 \
    -L 10005:localhost:10001 \
    -L 10004:localhost:8888 \
    -L 10022:localhost:22 \
    -N -f \
    username@<REMOTE_SERVER_IP>
```

**Replace**:

- `/path/to/your/ssh_private_key` → Your SSH private key path (e.g., `~/.ssh/id_rsa`)
- `username` → Your username on the remote server
- `<REMOTE_SERVER_IP>` → IP address of your remote server

**What happens**:[^4]

- `-L 10005:localhost:10001` → Forward local port 10005 to remote wb-svc service
- `-L 10004:localhost:8888` → Forward local port 10004 to remote proxy/Jupyter
- `-L 10022:localhost:22` → Forward local port 10022 to remote SSH (for Workbench verification)
- `-N -f` → Run tunnel in background without shell[^5]
- `-o ServerAliveInterval=15` → Send keepalive packets every 15 seconds
- `-o ServerAliveCountMax=4` → Close connection after 4 failed keepalives

***

## Step 5: Verify SSH Tunnel

**On your local machine**:

```bash
# Check if all tunnel ports are active (Unix/Mac/Linux)
lsof -i :10005
lsof -i :10004
lsof -i :10022

# For Windows, use:
# netstat -ano | findstr "10005"
# netstat -ano | findstr "10004"
# netstat -ano | findstr "10022"

# Test wb-svc service through tunnel
curl http://localhost:10005/v1/version
```

**Expected response**:

```json
{"built_on":"Thu Oct 30 17:38:11 UTC 2025","channel":"stable","version":"0.80.4"}
```

✅ If this works, your tunnel is fully operational!

***

## Step 6: Configure contexts.json

**On your local machine**:

```bash
# Create backup if file exists (Unix/Mac/Linux)
cp ~/.nvwb/contexts.json ~/.nvwb/contexts.json.backup

# For Windows:
# copy %USERPROFILE%\.nvwb\contexts.json %USERPROFILE%\.nvwb\contexts.json.backup

# Edit configuration file
nano ~/.nvwb/contexts.json

# For Windows, use Notepad:
# notepad %USERPROFILE%\.nvwb\contexts.json
```

**Complete file content** (replace entirely):[^2][^1]

```json
{
  "local": {
    "name": "local",
    "description": "My Local Computer",
    "hostname": "localhost",
    "workbenchDir": "/home/username/.nvwb",
    "contextType": "local",
    "contextTypeMetadata": {},
    "sshKeyPath": null,
    "sshUsername": null,
    "sshPort": null,
    "sshFingerprint": null,
    "proxyPort": 10000,
    "servicePort": 10001
  },
  "remote-direct": {
    "name": "remote-direct",
    "description": "Remote server direct SSH",
    "hostname": "<REMOTE_SERVER_IP>",
    "workbenchDir": "/home/username/.nvwb",
    "contextType": "manual",
    "contextTypeMetadata": {},
    "sshKeyPath": "/path/to/your/ssh_private_key",
    "sshUsername": "username",
    "sshPort": 22,
    "sshFingerprint": "SHA256:YOUR_SERVER_SSH_FINGERPRINT",
    "proxyPort": 10002,
    "servicePort": 10003
  },
  "remote-tunnel": {
    "name": "remote-tunnel",
    "description": "Remote via SSH Tunnel",
    "hostname": "127.0.0.1",
    "workbenchDir": "/home/username/.nvwb",
    "contextType": "manual",
    "contextTypeMetadata": {},
    "sshKeyPath": "/path/to/your/ssh_private_key",
    "sshUsername": "username",
    "sshPort": 10022,
    "sshFingerprint": "SHA256:YOUR_SERVER_SSH_FINGERPRINT",
    "proxyPort": 10004,
    "servicePort": 10005
  }
}
```

**Replace the following placeholders**:

1. **`<REMOTE_SERVER_IP>`** → Your remote server's IP address (e.g., `192.168.1.100`)
2. **`/home/username/.nvwb`** → Remote workbench directory path (keep as-is for Linux servers, or adjust for your setup)
3. **`/path/to/your/ssh_private_key`** → Full path to your SSH private key
    - Mac/Linux: `/home/username/.ssh/id_rsa`
    - Windows: `C:\Users\username\.ssh\id_rsa`
4. **`username`** → Your username on the remote server
5. **`SHA256:YOUR_SERVER_SSH_FINGERPRINT`** → SSH fingerprint of your remote server

### How to Get SSH Fingerprint

```bash
# On your local machine, connect to remote server and note the fingerprint
ssh-keyscan -t ed25519 <REMOTE_SERVER_IP> | ssh-keygen -lf -

# Or from an existing connection:
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

**Critical configuration details for `remote-tunnel`**:[^1]

- `"hostname": "127.0.0.1"` ← Must be `127.0.0.1`, NOT `"localhost"` (bypasses Workbench port validation)
- `"sshPort": 10022` ← Tunneled SSH port
- `"servicePort": 10005` ← Tunneled wb-svc port
- `"proxyPort": 10004` ← Tunneled proxy port
- `"contextType": "manual"` ← Must be `"manual"`, NOT `"local"`

Save the file: `Ctrl+O`, `Enter`, `Ctrl+X` (nano) or `Ctrl+S` (Notepad)

***

## Step 7: Restart NVIDIA AI Workbench Desktop App

**On your local machine**:

### Mac/Linux:

```bash
# Terminate the app
killall "NVIDIA AI Workbench"

# Clear cache
rm -rf ~/Library/Caches/NVIDIA\ AI\ Workbench/  # Mac
# or
rm -rf ~/.cache/NVIDIA\ AI\ Workbench/           # Linux

# Wait 2 seconds
sleep 2

# Restart the app
open -a "NVIDIA AI Workbench"  # Mac
# or
nvidia-ai-workbench  # Linux
```


### Windows:

```powershell
# Close app via Task Manager or:
taskkill /F /IM "NVIDIA AI Workbench.exe"

# Clear cache
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\NVIDIA\AI Workbench\Cache"

# Wait 2 seconds
Start-Sleep -Seconds 2

# Restart app from Start Menu or:
Start-Process "C:\Program Files\NVIDIA Corporation\NVIDIA AI Workbench\NVIDIA AI Workbench.exe"
```


***

## Step 8: Connect to remote-tunnel Location

**In the NVIDIA AI Workbench Desktop App**:

1. Open **Locations Manager**
2. You should see 3 locations:[^2]
    - `local` (Your local computer)
    - `remote-direct` (Direct SSH - may timeout over VPN)
    - `remote-tunnel` (Via SSH Tunnel) ✅
3. Click on **remote-tunnel**
4. If a fingerprint warning appears, **accept** it
5. The app should connect successfully! 🎉

***

## Daily Workflow

### Morning Setup Routine

**1. Connect VPN**

```bash
# Start your VPN client
# Verify connectivity:
ping -c 3 <REMOTE_SERVER_IP>
```

**2. Connect VSCode to Remote** (Optional, for verification)

```bash
# VSCode → Cmd/Ctrl+Shift+P
# "Remote-SSH: Connect to Host" → Select your server
```

**3. Start wb-svc on Remote Server**
**In VSCode Terminal or direct SSH**:

```bash
~/.nvwb/bin/wb-svc serve &
```

**4. Start SSH Tunnel on Local Machine**

```bash
ssh -i "/path/to/your/ssh_private_key" \
    -o ServerAliveInterval=15 \
    -o ServerAliveCountMax=4 \
    -L 10005:localhost:10001 \
    -L 10004:localhost:8888 \
    -L 10022:localhost:22 \
    -N -f \
    username@<REMOTE_SERVER_IP>
```

**5. Verify Tunnel**

```bash
curl http://localhost:10005/v1/version
```

**6. Open NVIDIA AI Workbench Desktop App**

```bash
# Mac/Linux
open -a "NVIDIA AI Workbench"

# Windows: Use Start Menu or desktop shortcut
```

**7. Select remote-tunnel location and start working!** ✅

***

## Troubleshooting

### Problem: "Address already in use"

**Solution**: Kill existing tunnel processes

```bash
# Unix/Mac/Linux
lsof -i :10005
kill -9 <PID>

lsof -i :10004
kill -9 <PID>

lsof -i :10022
kill -9 <PID>

# Windows
netstat -ano | findstr "10005"
taskkill /F /PID <PID>

# Restart tunnel (repeat Step 4)
```


### Problem: "Connection reset by peer" when testing curl

**Solution**: wb-svc is not running on remote server

```bash
# SSH to remote server or use VSCode Terminal
ps aux | grep wb-svc
curl http://localhost:10001/v1/version

# If not running:
~/.nvwb/bin/wb-svc serve &
```


### Problem: "SSH timeout" when creating tunnel

**Solution**: VPN connection issue

```bash
# Check VPN connectivity
ping -c 3 <REMOTE_SERVER_IP>

# If timeout: Reconnect VPN
# Ensure VPN routes include your remote server's subnet
```


### Problem: "Failed to get context SSH fingerprints"

**Solution**: Deactivate context and restart app[^1]

```bash
# Deactivate context via CLI
~/.nvwb/bin/nvwb deactivate remote-tunnel --shutdown

# Restart app
killall "NVIDIA AI Workbench"
rm -rf ~/Library/Caches/NVIDIA\ AI\ Workbench/  # Mac/Linux
open -a "NVIDIA AI Workbench"
```


### Problem: Tunnel disconnects after 1-2 minutes

**Solution A: Use autossh for automatic reconnection** (Recommended)

```bash
# Install autossh
# Mac: brew install autossh
# Linux: sudo apt install autossh
# Windows: Use Windows Subsystem for Linux (WSL)

# Start auto-reconnecting tunnel
autossh -M 0 \
    -o "ServerAliveInterval 30" \
    -o "ServerAliveCountMax 3" \
    -o "ExitOnForwardFailure=yes" \
    -i "/path/to/your/ssh_private_key" \
    -L 10005:localhost:10001 \
    -L 10004:localhost:8888 \
    -L 10022:localhost:22 \
    -N -f \
    username@<REMOTE_SERVER_IP>
```

**Solution B: Create manual reconnection script**

Create file: `tunnel-auto-reconnect.sh`

```bash
#!/bin/bash
while true; do
  echo "$(date): Starting tunnel..."
  ssh -i "/path/to/your/ssh_private_key" \
      -o ServerAliveInterval=15 \
      -o ServerAliveCountMax=4 \
      -o ExitOnForwardFailure=yes \
      -L 10005:localhost:10001 \
      -L 10004:localhost:8888 \
      -L 10022:localhost:22 \
      -N \
      username@<REMOTE_SERVER_IP>
  
  echo "$(date): Tunnel disconnected, reconnecting in 5 seconds..."
  sleep 5
done
```

Make executable and run:

```bash
chmod +x tunnel-auto-reconnect.sh
./tunnel-auto-reconnect.sh &
```


### Problem: Invalid local context error

**Error**: "the proxy port and service port must be 10000 and 10001"

**Solution**: Ensure `hostname` is set to `"127.0.0.1"` (not `"localhost"`) in `contexts.json`[^1]

***

## Port Reference Table

| Local Port | Remote Port | Purpose |
| :-- | :-- | :-- |
| 10005 | 10001 | wb-svc Service API [^1] |
| 10004 | 8888 | Proxy/Jupyter for applications [^2] |
| 10022 | 22 | SSH for fingerprint verification |


***

## Summary: What Works

✅ **VPN Connection** - Base connectivity to remote server
✅ **SSH Key Authentication** - Required by Workbench[^1]
✅ **3-Port SSH Tunnel** - Service (10005), Proxy (10004), SSH (10022)
✅ **contexts.json Configuration** - remote-tunnel context with hostname: 127.0.0.1
✅ **Desktop App** - Uses remote-tunnel location through tunnel
✅ **wb-svc Service** - Runs on remote server port 10001

## Limitations

⚠️ **Tunnel may disconnect** - VPN keepalive issues (use autossh for auto-reconnection)
⚠️ **Manual setup required** - Daily VPN + Tunnel + wb-svc startup needed
⚠️ **VSCode more stable** - Consider using VSCode Remote-SSH for longer development sessions

***

## Additional Resources

- [NVIDIA AI Workbench Remote Locations Documentation](https://docs.nvidia.com/ai-workbench/user-guide/latest/reference/remote-locations.html)[^1]
- [SSH Tunneling Best Practices](https://security.berkeley.edu/education-awareness/securing-network-traffic-ssh-tunnels)[^4]
- [Remote Development Setup Guide](https://www.designmind.com/blog/cloud-computing/setting-up-a-remote-development-environment)[^6]

***

**This configuration provides a reliable workaround for accessing NVIDIA AI Workbench on remote GPU servers through unstable VPN connections!** 🚀
<span style="display:none">[^10][^7][^8][^9]</span>

<div align="center">⁂</div>

[^1]: https://docs.nvidia.com/ai-workbench/user-guide/latest/reference/remote-locations.html

[^2]: https://docs.nvidia.com/ai-workbench/user-guide/latest/reference/faqs/location-faq.html

[^3]: https://www.youtube.com/watch?v=Unr8VWZIaRk

[^4]: https://security.berkeley.edu/education-awareness/securing-network-traffic-ssh-tunnels

[^5]: https://specterops.io/blog/2021/04/22/offensive-security-guide-to-ssh-tunnels-and-proxies/

[^6]: https://www.designmind.com/blog/cloud-computing/setting-up-a-remote-development-environment

[^7]: https://docs.rapids.ai/deployment/stable/platforms/nvidia-ai-workbench/

[^8]: https://github.com/NVIDIA/nim-anywhere

[^9]: https://www.aurelio.ai/learn/ai-workbench-remote

[^10]: https://engineerworkshop.com/blog/how-to-set-up-a-remote-development-server/
