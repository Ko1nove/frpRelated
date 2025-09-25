# Remote Desktop via frp

## Architecture Diagram

Below is a simplified view of how frp tunnels RDP traffic:

```

[ Windows Client (frpc) ]                 [ Public Server (frps) ]              [ RD Client App ]
+-----------------------+                 +----------------------+              +-------------------+
| frpc.exe              |   TCP:7000 ---> | frps                 |              | Connects to:      |
|                       |                 |                      |   TCP:33389  |                   |
| Local RDP (3389)      | <---------------| Forward to local:3389|------------->| 8.8.8.8:33389     |
+-----------------------+                 +----------------------+              +-------------------+

Key:

* Port 3389: Windows built-in Remote Desktop (local only)
* Port 7000: frpc ↔ frps control channel
* Port 33389: Publicly exposed RDP access via frp
```

---

## Requirements
- A cloud server with a public IP (e.g., Alibaba Cloud, Baidu Cloud, Tencent Cloud, etc.)  
- [frp](https://github.com/fatedier/frp)

---

## Setup Guide

### 1. Server Side (Public Server)
Keep `frps` binary and `frps.toml` configuration file on the server.

Example `frps.toml`:
```toml
bindAddr = "0.0.0.0"
bindPort = 7000
````

Then start `frps`:

```bash
./frps -c frps.toml
```

---

### 2. Client Side (Personal Windows Device)

Keep `frpc.exe` and `frpc.toml` on your Windows machine.
Edit `frpc.toml` and configure it to actively connect to your public server.

Example `frpc.toml`:

```toml
serverAddr = "YOUR_PUBLIC_SERVER_IP"
serverPort = 7000

[[proxies]]
name = "RDremote"
type = "tcp"
localIP = "127.0.0.1"
localPort = 3389
remotePort = 33389
```

Then start `frpc`:

```powershell
frpc.exe -c frpc.toml
```

---

### 3. Connect via Microsoft RD Client

In your RD Client (Windows App Mobile):

* Use your public IP and mapped port as the connection address.

For example, if your server’s public IP is `8.8.8.8`,
then connect to:

```
8.8.8.8:33389
```

---

## Firewall Configuration

### On the Server (Public Cloud Host)
You need to allow the following ports in your firewall/security group:

- **7000/tcp** → for frps to accept client connections  
- **33389/tcp** → mapped RDP port (from frpc to actual Windows RDP 3389)

Example (Linux server using `ufw`):
```bash
sudo ufw allow 7000/tcp
sudo ufw allow 33389/tcp
````

If you are using a cloud provider (e.g., Alibaba Cloud, Tencent Cloud), make sure to add the same inbound rules in the provider’s **security group**.

---

### On the Client (Windows PC)

1. **Inbound Rules**

   * RDP service normally listens on **3389/tcp**.
   * If you only connect via frp, you do **not** need to expose 3389 to the public network.
   * Just ensure Windows Firewall allows inbound connections to `3389/tcp` on **Local IP 127.0.0.1** (this is usually enabled by default when Remote Desktop is turned on).

2. **Outbound Rules**

   * `frpc.exe` must be able to connect out to your server on:

     * **7000/tcp** (frps port)
   * Windows Firewall typically allows outbound by default, but you can explicitly add a rule if needed.

---

### Quick Checklist

* ✅ Server: `7000/tcp` open
* ✅ Server: `33389/tcp` open
* ✅ Client: Remote Desktop enabled (3389 locally accessible)
* ✅ Client: Outbound rule allows `frpc.exe` to connect on 7000/tcp

