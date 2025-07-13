# 🤖 n8n + Telegram Bot Integration with ngrok (Self-Hosted on Ubuntu VM)

This project documents how I set up **n8n**, an open-source workflow automation tool, to work with a **Telegram bot** using **ngrok** for public webhook access. Everything runs self-hosted inside **Docker containers** on an **Ubuntu Server VM** (hosted in VMware Workstation bridged to LAN) on a **Windows 11 Education Edition** machine. The key idea is to trigger workflows in n8n based on incoming Telegram messages, even when the setup is entirely behind NAT/firewall, without static IP or HTTPS certs. All traffic is tunneled securely via ngrok.

---

## 📺 YouTube Tutorial

<img width="1165" height="705" alt="image" src="https://github.com/user-attachments/assets/5ed84330-8119-47f9-b8ab-877a61ae8a51" />

I recorded a full walkthrough of this setup. You can follow along with the video here:  
🎬 [https://www.youtube.com/watch?v=NmcVa_g9kzc](https://www.youtube.com/watch?v=NmcVa_g9kzc)

---

## 🧩 System Architecture

This stack consists of multiple containers:
- **n8n**: The main automation engine running at port `5678`
- **ngrok**: Exposes n8n’s local port securely over HTTPS
- **Portainer**: (Optional) Docker management UI
- **WatchYourLAN**: (Optional) Network scanner

All containers are connected via a custom Docker network called `n8n-net`. This allows internal DNS resolution, so n8n can refer to `http://ollama:11434` or `http://ngrok:4040` seamlessly.

---

## ⚙️ Workflow Concept

The setup allows you to:
- Create Telegram-triggered workflows inside n8n
- Filter messages based on chat ID (to isolate group vs private messages)
- Respond with logic, notifications, or integrations (e.g., APIs, emails, AI, etc.)

When a message is sent to your Telegram bot (in a group or private), Telegram sends it to the webhook URL that n8n registers. Since n8n runs locally, the webhook wouldn't normally be reachable — that’s where **ngrok** comes in, exposing it over a public HTTPS URL.

---

## 🛠️ Installation & Setup

### Step 1: Create a Docker Network

This ensures all containers can communicate by name:

```bash
docker network create n8n-net
````

---

### Step 2: Run ngrok Container

This exposes `n8n:5678` to the public via HTTPS. Replace the `NGROK_AUTHTOKEN` with your token from [https://dashboard.ngrok.com](https://dashboard.ngrok.com):

```bash
docker run -d \
  --name ngrok \
  --network n8n-net \
  -p 4040:4040 \
  -e NGROK_AUTHTOKEN=your_ngrok_token \
  ngrok/ngrok http n8n:5678
```

You can access the ngrok web UI at:
📡 `http://localhost:4040`

---

### Step 3: Get Your ngrok Public URL

After the container starts, get the public URL that ngrok assigned:

```bash
curl http://localhost:4040/api/tunnels | jq -r '.tunnels[0].public_url'
```

You’ll get something like:

```
https://143f02d83119.ngrok-free.app
```

---

### Step 4: Run n8n Docker Container

Now that you have the ngrok HTTPS URL, launch `n8n` with it passed as environment variables:

```bash
docker rm -f n8n

docker run -d \
  --name n8n \
  --network n8n-net \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  -e N8N_HOST=n8n \
  -e N8N_PORT=5678 \
  -e N8N_PROTOCOL=http \
  -e WEBHOOK_URL=https://143f02d83119.ngrok-free.app \
  -e WEBHOOK_TUNNEL_URL=https://143f02d83119.ngrok-free.app \
  -e N8N_SECURE_COOKIE=false \
  n8nio/n8n
```

This ensures that n8n uses your ngrok public address when registering webhooks (e.g., with Telegram).

---

### Step 5: Create Telegram Bot

1. Open Telegram and talk to [@BotFather](https://t.me/BotFather)
2. Use `/newbot` and follow the instructions
3. Copy the **bot token** shown at the end

---

### Step 6: Add Bot to Telegram Group

If you want your bot to respond to messages in a private group:

1. Add the bot to the group like any other member
2. Promote it to **admin**
3. Send a message in the group to generate activity

---

### Step 7: Find the Group Chat ID

You can find the group chat ID using one of two ways:

* Use the Telegram Trigger node and inspect the incoming message JSON
* Or visit:

```bash
https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates
```

Look for a line like:

```json
"chat": {
  "id": -1001234567890,
  ...
}
```

---

### Step 8: Create the Workflow in n8n

1. Open n8n via `https://<your-ngrok-url>`
2. Add a **Telegram Trigger** node
3. Use your bot token in the credentials
4. Add an **IF** node:

   * Condition: `{{$json["message"]["chat"]["id"]}} is equal to -1001234567890`
5. Optionally, add a **Set** node to process or reply
6. Save and activate the workflow

---

## 🧪 Testing

* Click **"Execute Node"** in the Telegram Trigger
* Send a message in your group
* The workflow should trigger if the chat ID matches
* The message will pass through the IF node and continue the flow

---

## 🛠️ Optional Tools

### WatchYourLAN (LAN IP monitoring)

```bash
docker run -d \
  --name watchyourlan \
  --network host \
  -e TZ="Asia/Kolkata" \
  -e WEB_USER=admin \
  -e WEB_PASS=pass \
  aceberg/watchyourlan
```

### Portainer (Docker UI)

```bash
docker run -d \
  --name portainer \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  portainer/portainer-ce
```

---

## 🧯 Troubleshooting

### Telegram Trigger gives HTTPS error?

Make sure `WEBHOOK_URL` is set to your ngrok **HTTPS** address, not `localhost`.

### n8n gives "secure cookie" warning?

Set `N8N_SECURE_COOKIE=false` to avoid cookie problems through the tunnel.

### Workflow doesn’t trigger?

Check:

* Bot has admin rights
* Chat ID in IF node is correct
* ngrok tunnel is live
* ngrok URL hasn't changed (it does on restart)

---

## 📌 Notes

* This setup is ideal for quick automation testing or internal bots
* For production use, consider a reverse proxy like Caddy with a domain + TLS
* You can add AI (LLMs like OpenRouter, Ollama) via HTTP nodes for chatbot use cases

---

## 🙋 Author

Built by **@flatmarstheory**
Inspired by self-hosting simplicity, Docker networking, and no-frills automation.
📺 YouTube tutorial: [https://www.youtube.com/watch?v=NmcVa\_g9kzc](https://www.youtube.com/watch?v=NmcVa_g9kzc)

---

## 📄 License

This project is released under the MIT License.
Feel free to fork, modify, and contribute.
