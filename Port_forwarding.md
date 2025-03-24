# Port Forwarding Manual

## Ngrok Port Forwarding

### 1. Installing and Setting Up Ngrok
Ngrok is a tunneling tool that allows secure exposure of local servers to the internet.

#### **Step 1: Download and Install Ngrok**
```bash
wget https://bin.equinox.io/c/bNyj1mQVY4c/ngrok-stable-linux-amd64.zip
unzip ngrok-stable-linux-amd64.zip
sudo mv ngrok /usr/local/bin/
```

#### **Step 2: Authenticate Ngrok**
Sign up at [Ngrok's website](https://ngrok.com/) and get an authentication token.
```bash
ngrok authtoken YOUR_AUTH_TOKEN
```

#### **Step 3: Forward Ports**
- **Expose a Web Server (HTTP 80):**
```bash
ngrok http 80
```
- **Expose SSH (TCP 22):**
```bash
ngrok tcp 22
```
- **Expose MongoDB (TCP 27017):**
```bash
ngrok tcp 27017
```

#### **Step 4: Secure Usage**
- Use authentication headers for security.
- Restrict IP access using Ngrok’s dashboard.
- Use `ngrok http --auth="user:password" 80` for basic authentication.

---

## Cloudflared Port Forwarding

### 1. Installing and Setting Up Cloudflared
Cloudflared provides secure tunneling for Cloudflare services.

#### **Step 1: Download and Install Cloudflared**
```bash
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64
chmod +x cloudflared-linux-amd64
sudo mv cloudflared-linux-amd64 /usr/local/bin/cloudflared
```

#### **Step 2: Authenticate Cloudflared**
Sign in to Cloudflare and get a token from [Cloudflare Tunnel Dashboard](https://dash.teams.cloudflare.com/).
```bash
cloudflared tunnel login
```

#### **Step 3: Create and Run a Tunnel**
- **Create a Tunnel:**
```bash
cloudflared tunnel create my-tunnel
```
- **Expose a Web Server (HTTP 80):**
```bash
cloudflared tunnel route dns my-tunnel example.com
cloudflared tunnel run my-tunnel
```
- **Expose SSH (TCP 22):**
```bash
cloudflared access ssh --hostname ssh.example.com --url ssh://localhost:22
```

#### **Step 4: Secure Usage**
- Enable Cloudflare Access for authentication.
- Restrict access via Cloudflare’s Zero Trust settings.
- Use `cloudflared tunnel --metrics localhost:9090` to monitor traffic.

This manual covers detailed steps for safely forwarding ports using Ngrok and Cloudflared. Adjust security settings based on your requirements.

