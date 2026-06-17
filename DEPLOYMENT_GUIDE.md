# Deployment Guide: Host the RAG Chatbot on Render (Free)

This guide walks you through deploying the Conversation RAG Chatbot to the cloud so anyone can access it via a public URL.

---

## Step 1: Push Code to GitHub

### 1.1 Create a GitHub repository
- Go to https://github.com/new
- Name it `rag-chatbot`
- Choose **Public**
- Click **Create repository**

### 1.2 Push your local code

```bash
cd /path/to/your/rag-chatbot-project

git init
git add .
git commit -m "Initial commit"

git remote add origin https://github.com/YOUR_USERNAME/rag-chatbot.git
git push -u origin main
```

**Important**: Include the pre-built index folder (it's ~300MB):
```bash
git add tasks/rag_system/index/
git push
```

---

## Step 2: Sign Up on Render (Free)

1. Go to https://render.com
2. Click **Get Started for Free**
3. Sign up with your **GitHub** account

---

## Step 3: Deploy the Web Service

1. On Render dashboard, click **New** → **Web Service**
2. Connect your `rag-chatbot` GitHub repository
3. Configure these settings:

| Setting | Value |
|---------|-------|
| **Name** | `rag-chatbot` |
| **Environment** | Python 3 |
| **Region** | Oregon (US West) — closest to you |
| **Branch** | `main` |
| **Root Directory** | `tasks` |
| **Build Command** | `pip install -r requirements.txt` |
| **Start Command** | `gunicorn rag_system.chatbot:app` |
| **Plan** | Free |

4. Click **Create Web Service**

---

## Step 4: Wait for Build & Deploy

Render will:
1. Install dependencies from `requirements.txt`
2. Start the Flask app via Gunicorn
3. Assign a public URL

**Build time**: ~2–5 minutes

You will see a URL like:
```
https://rag-chatbot-abc123.onrender.com
```

---

## Step 5: Test Your Deployed Chatbot

Open the URL in your browser. You should see:
- The **Conversation RAG Chatbot** title
- The **User Persona** card
- The **chat interface** where you can ask questions

Try these test queries:
- `What kind of person is this user?`
- `What are their habits?`
- `Tell me about Portland`

Also test the API directly:
```bash
curl https://YOUR_URL.onrender.com/persona
curl -X POST https://YOUR_URL.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"query":"What are their habits?"}'
```

---

## Step 6: Share the URL with Evaluators

Copy the URL from your Render dashboard and share it. Evaluators can:
- Open it in any browser
- Use the chat interface interactively
- Call API endpoints directly

---

## Troubleshooting

### Port already in use error
This shouldn't happen on Render, but if you see it locally, set the PORT environment variable:
```bash
export PORT=5000
```

### Build fails
Check that `requirements.txt` and `Procfile` are in the `tasks/` directory:
```
tasks/
├── requirements.txt
├── Procfile
└── rag_system/
    └── chatbot.py
```

### App loads slowly on first visit (Free tier)
Render free tier spins down after 15 minutes of inactivity. First visit after inactivity may take 30–60 seconds to wake up. This is normal.

### Large file size (index folder)
The `rag_system/index/` folder contains ~300MB of pre-built TF-IDF vectors. Two options:

**Option A (Recommended):** Include it in GitHub push → app loads instantly.

**Option B:** Add `rag_system/index/` to `.gitignore`, let Render build the index on first startup (takes ~30 seconds).

---

## Alternative: Deploy to Railway (Also Free)

1. Go to https://railway.app, sign up with GitHub
2. Click **New Project** → **Deploy from GitHub**
3. Select your `rag-chatbot` repo
4. Set **Start Command**: `gunicorn --chdir tasks rag_system.chatbot:app`
5. Done — Railway gives you a public URL immediately

---

## Alternative: Deploy to PythonAnywhere (Free)

1. Go to https://www.pythonanywhere.com, sign up
2. Upload files via the Files tab
3. Configure WSGI for Flask
4. Set `app = 'rag_system.chatbot:app'`
5. Reload web app → get `yourname.pythonanywhere.com` URL

---

## Summary Checklist

- [ ] Push code to GitHub (including `index/` folder)
- [ ] Sign up on Render with GitHub
- [ ] Create New Web Service
- [ ] Set Build Command: `pip install -r requirements.txt`
- [ ] Set Start Command: `gunicorn rag_system.chatbot:app`
- [ ] Set Root Directory: `tasks`
- [ ] Deploy
- [ ] Copy public URL
- [ ] Share URL with evaluators
