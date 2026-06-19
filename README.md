
भाई, आपके इस पूरे ऑल-इन-वन आर्किटेक्चर (Bifrost + Redis + Qdrant) के लिए एक बेहद प्रोफेशनल और स्टेप-बाय-स्टेप **`README.md`** फाइल तैयार कर दी है।

इसे आप सीधे कॉपी करके अपने गिटहब रिपोजिटरी में डाल सकते हैं, ताकि भविष्य में आपको या आपकी टीम को दोबारा सेटअप करने में सिर्फ 2 मिनट का समय लगे।

```markdown
# Bifrost All-in-One Stack Deployment Guide

This repository contains the production-ready **Docker Compose** configuration to deploy **Bifrost (by Maxim AI)** alongside **Redis Stack** (for Semantic Caching) and **Qdrant** (for Vector Store/RAG) seamlessly inside **Coolify**.

---

## 🚀 Architecture Overview
- **Bifrost:** Main LLM Gateway & Application Router.
- **Redis Stack:** High-performance, in-memory database used for **Semantic Caching** to eliminate duplicate LLM API costs and achieve sub-millisecond latency.
- **Qdrant:** Highly optimized vector database dedicated to **Skills Repository / Knowledge Base (RAG)** to save server RAM and manage large document lookups.
- **Networking:** Unified under Coolify's native `coolify` network, enabling clean internal DNS resolution (`redis:6379`) without random UUID hostname collision.

---

## 🛠️ Step-by-Step Deployment Instructions

### Step 1: Add Docker Compose Stack in Coolify
1. Go to your Coolify Dashboard and create a new **Docker Compose** based application.
2. Name the stack (e.g., `Bifrost-Production-Stack`).
3. Paste the following configuration into the Compose box and click **Save**:

```yaml
version: '3.8'
services:
  bifrost:
    image: 'maximhq/bifrost:latest'
    ports:
      - '8080:8080'
    volumes:
      - './bifrost_config.json:/app/data/config.json'
    depends_on:
      - redis
      - qdrant
    restart: unless-stopped
    networks:
      - coolify

  redis:
    image: 'redis/redis-stack:latest'
    ports:
      - '6379:6379'
    volumes:
      - 'redis_data:/data'
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - coolify

  qdrant:
    image: 'qdrant/qdrant:latest'
    ports:
      - '6333:6333'
      - '6334:6334'
    volumes:
      - 'qdrant_data:/qdrant/storage'
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/readyz"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - coolify

networks:
  coolify:
    external: true

volumes:
  redis_data:
    driver: local
  qdrant_data:
    driver: local

```

---

### Step 2: Configure `bifrost_config.json` (Persistent Storage)

To lock down performance settings and map the semantic cache engine, you must create a local configuration file within Coolify.

1. In the same stack page, navigate to the **Storages** / **Files** tab.
2. Click **Add File** / **Create File**.
3. Set the **File Path** exactly to: `bifrost_config.json`
4. Paste the following JSON content to set optimized pool limits and internal routing:

```json
{
  "source_of_truth": "config.json",
  "client": {
    "initial_pool_size": 100,
    "max_request_body_size_mb": 100
  },
  "vector_store": {
    "enabled": true,
    "type": "redis",
    "config": {
      "addr": "redis:6379"
    }
  }
}

```

5. Click **Save Changes**.

---

### Step 3: Performance Tuning & LLM Provider Setup (Bifrost UI)

Once the stack is successfully **Deployed** and status shows `🟢 Running (healthy)`, log in to your Bifrost Web UI (`https://your-domain.com`) and apply these final baseline configurations:

#### 1. Caching Composition Settings

* Go to **Settings** ➡️ **Caching**.
* Verify that the red warning block is gone and the configurations are unlocked.
* **Enable Caching:** Toggle to **ON**.
* **Cache by Model:** Turn **ON** (Ensures separate cached pools for different LLMs like GPT-4 vs Claude).
* **Cache by Provider:** Turn **ON**.

#### 2. Provider Network Configuration (Fixing Custom API Errors)

If using custom base URLs or third-party proxy gateways (e.g., OpenRouter, DeepSeek, Freemodel):

* Go to **Models / Providers** ➡️ **OpenAI (or Custom Provider)** ➡️ **Edit Provider Config**.
* **Max Connections Per Host:** Change from `5000` to **`100`** (Prevents the proxy provider from flagging your requests as a DDoS attack).
* **Initial Backoff (ms):** Set to `500`.
* **Max Backoff (ms):** Set to `5000` (Fixes validation errors where Initial Backoff must be less than Max Backoff).
* Ensure your custom endpoint link is properly appended in the **Base URL** box.
* Click **Save Network Configuration**.

---

## ⚡ Verification & Testing

To confirm semantic caching is operational:

1. Open the Bifrost Chat Playground and submit a prompt (e.g., `Explain Quantum Computing in one sentence`). Note the initial latency.
2. Open a completely new session or tab and submit the **exact same prompt**.
3. The response should render instantaneously (**0-10ms** execution time), confirming that the background **Redis Vector Store** is successfully routing cached hits.

```

```
