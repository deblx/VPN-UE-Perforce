# 🎮 Unreal Engine + Perforce (Helix Core) + WireGuard VPN Setup

**Guide to setting up Unreal Engine collaboration with Perforce (Helix Core) over a WireGuard VPN, using an Ubuntu/Debian server and Windows clients.**

A complete guide to setting up a secure collaborative workflow for Unreal Engine development using:

- ✅ **Ubuntu/Debian Server** (Perforce + WireGuard)
- ✅ **Windows Clients** (Unreal Engine + P4V)
- ✅ **Secure LAN-like connectivity** via WireGuard VPN


---

## 🔒 Security & Licensing Notice

- Make sure to **secure your server**:
  - Use strong SSH passwords or key authentication
  - Only expose necessary ports (e.g., VPN and Perforce)
  - Restrict access to VPN IPs via firewall

- Regularly update your system and packages to avoid vulnerabilities.

- ✅ **Perforce (Helix Core Server)** Please review the Perforce licensing terms.  

> 🚨 Always follow best practices for securing infrastructure, especially if hosting remotely.


---

## ⚙️ 1. Setup WireGuard VPN

### 🖥️ On Ubuntu/Debian Server

```bash
sudo apt update
sudo apt install wireguard
wg genkey | tee privatekey | wg pubkey > publickey
```

Create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.66.66.1/24
PrivateKey = <server-private-key>
ListenPort = 51820
SaveConfig = true
```

Enable IP forwarding:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Start the VPN:

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

---

### 💻 On Windows Clients

Install [WireGuard for Windows](https://www.wireguard.com/install/) and use:

```ini
[Interface]
PrivateKey = <client-private-key>
Address = 10.66.66.2/32

[Peer]
PublicKey = <server-public-key>
Endpoint = your.server.ip:51820
AllowedIPs = 10.66.66.0/24
PersistentKeepalive = 25
```

> ✅ Test: `ping 10.66.66.1`

---

## 📦 2. Install Perforce (Helix Core Server)

### On Ubuntu/Debian Server

```bash
wget -qO - https://package.perforce.com/perforce.pubkey | sudo apt-key add -
echo "deb http://package.perforce.com/apt/ubuntu jammy release" | sudo tee /etc/apt/sources.list.d/perforce.list
sudo apt update
sudo apt install -y helix-p4d
```

Create required directories:

```bash
sudo mkdir -p /opt/perforce/metadata /opt/perforce/depot
sudo useradd -r -s /bin/false perforce || true
sudo chown -R perforce: /opt/perforce
```

Initialize the database:

```bash
sudo -u perforce /usr/sbin/p4d -r /opt/perforce/metadata -J journal -xi
```

Create systemd service:

```ini
# /etc/systemd/system/perforce.service

[Unit]
Description=Helix Core Perforce Server
After=network.target

[Service]
User=perforce
ExecStart=/usr/sbin/p4d -r /opt/perforce/metadata -p 1666 -L /opt/perforce/metadata/log -J /opt/perforce/metadata/journal
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reexec
sudo systemctl enable perforce
sudo systemctl start perforce
```

Install CLI (optional):

```bash
wget https://cdist2.perforce.com/perforce/r24.1/bin.linux26x86_64/p4
chmod +x p4
sudo mv p4 /usr/local/bin/
```

---

## 👤 3. Create User, Depot, and Workspace

### On Ubuntu/Debian Server

```bash
p4 -p 127.0.0.1:1666 user
p4 passwd
p4 depot UEProject
```

### In P4V (Windows)

- Workspace Name: `client1_ue`
- Root: `D:\UnrealProjects\MyGame`
- View:
  ```
  //UEProject/... //client1_ue/...
  ```


## 🖥️ P4V: Installing & Using the Perforce GUI

### 🧩 What is P4V?

**P4V** is the official graphical client for Perforce. It allows users to:

- Create and manage workspaces
- View changelists and depot structure
- Add, check out, and submit files
- Visualize file history and diffs

---

## 🛠️ Installing P4V on Windows

1. Download P4V from the official Perforce website:  
   👉 [https://www.perforce.com/downloads/helix-visual-client-p4v](https://www.perforce.com/downloads/helix-visual-client-p4v)

2. Install and launch the client.

3. On first launch, provide your server info:

| Field     | Example                  |
|-----------|---------------------------|
| Server    | `10.66.66.1:1666`        |
| Username  | your Perforce user        |
| Workspace | *(leave blank initially)* |

---

### ⚠️ Known Issue: Large Unreal Projects May Cause Freezes

When using the **"Add Files Wizard"** in P4V on large Unreal projects:

- P4V may **freeze or crash** ("Not Responding") due to scanning thousands of files
- Especially common if temporary folders like `Binaries/`, `Intermediate/`, or `Saved/` are included

---

### ✅ Safer Alternative: Use `p4` CLI for Initial Add

If P4V becomes unresponsive:

1. Open a terminal (PowerShell or CMD)
2. Set environment variables:

```powershell
p4 set P4PORT=10.66.66.1:1666
p4 set P4USER=your_username
p4 set P4CLIENT=your_workspace
```

---

## 🧩 4. Unreal Engine Integration

1. Open the `.uproject` inside the workspace root
2. Go to `File → Source Control → Connect to Source Control`
3. Choose `Perforce` and fill:

| Field     | Value                 |
|-----------|------------------------|
| Server    | `10.66.66.1:1666`     |
| Username  | Your Perforce user     |
| Workspace | `client1_ue`          |

4. Accept → green check ✅ appears

5. Add files:
   - ✅ `.uproject`, `Config/`, `Content/`, `Source/`
   - ❌ Skip: `Binaries/`, `Intermediate/`, `Saved/`

6. Submit:
```
File → Submit to Source Control
```

---

## 📄 Recommended `.p4ignore`

Place in project root:

```gitignore
Binaries/
DerivedDataCache/
Intermediate/
Saved/
.vs/
.vscode/
*.sdf
*.suo
*.user
*.log
```

Then:

```bash
p4 add .p4ignore
p4 submit -d "Add ignore file"
```

---

## ✅ You're Ready!

You now have:

- 🔒 VPN access to Perforce
- 🛠️ Source controlled Unreal Engine project
- 📡 Remote collaboration with low-latency file access

---

## 🔁 Workflow Tips

| Task                 | Action                              |
|----------------------|--------------------------------------|
| Check out file       | Right-click → Check Out              |
| Add file             | Right-click → Add to Source Control |
| Submit changes       | File → Submit to Source Control      |
| Discard changes      | Right-click → Revert                 |
| Sync latest changes  | File → Sync                          |


---

Made with ❤️ by Deblx for teams building games.
Like this? ⭐ Star & follow!

