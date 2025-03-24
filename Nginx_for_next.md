# Hosting a Next.js Project with Nginx on Kali Linux

This guide explains how to set up Nginx as a reverse proxy for your Next.js application, making it accessible on your **local network**.

---

## **1. Install and Start Nginx**
If Nginx is not installed, install it using:
```bash
sudo apt update && sudo apt install nginx -y
```
Start and enable Nginx to run on boot:
```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

---

## **2. Configure Nginx for Next.js**
Create a new configuration file for your Next.js app:
```bash
sudo nano /etc/nginx/sites-available/nextjs
```

### **Add the following configuration:**
```nginx
server {
    listen 80;
    server_name your-kali-ip;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```
- Replace `your-kali-ip` with your actual **local network IP** (e.g., `192.168.1.100`).

---

## **3. Enable the Configuration**
Create a symbolic link to activate the configuration:
```bash
sudo ln -s /etc/nginx/sites-available/nextjs /etc/nginx/sites-enabled/
```

Check for syntax errors:
```bash
sudo nginx -t
```

If everything is fine, restart Nginx:
```bash
sudo systemctl restart nginx
```

---

## **4. Stop Apache (If Running)**
If you see the default Apache page, stop Apache:
```bash
sudo systemctl stop apache2
sudo systemctl disable apache2
```
Then restart Nginx:
```bash
sudo systemctl restart nginx
```

---

## **5. Run Your Next.js Project**
Start your Next.js application:
```bash
npm run build && npm start
```
This will run the project on `localhost:3000`, which Nginx will serve via port `80`.

---

## **6. Access from Other Devices**
Find your Kali Linux local IP:
```bash
ip a
```
Look for an address like `192.168.1.100`. Now, on another device in your network, open a browser and go to:
```
http://192.168.1.100
```
Your Next.js app should be accessible across your local network! 🚀

---

## **7. Troubleshooting**
### **Check Nginx Status**
```bash
sudo systemctl status nginx
```
### **Check Nginx Logs**
```bash
sudo journalctl -u nginx --no-pager
```
### **Reload Nginx After Changes**
```bash
sudo systemctl reload nginx

