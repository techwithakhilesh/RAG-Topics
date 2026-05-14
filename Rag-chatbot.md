# 🛒 RAG E-Commerce Chatbot using LangChain + OpenAI + Node.js

A production-ready Retrieval-Augmented Generation (RAG) chatbot for an eCommerce website using:

* LangChain
* OpenAI
* Node.js
* Express.js
* Pinecone Vector Database
* Website Knowledge Base

This chatbot can:

✅ Answer product-related questions
✅ Search website knowledge base
✅ Recommend products
✅ Answer FAQs
✅ Handle return/shipping policies
✅ Use real business data instead of hallucinating

---

# 📌 Architecture Overview

```txt
User Question
      ↓
Frontend Chat UI
      ↓
Backend API (Node.js)
      ↓
Convert Query → Embedding
      ↓
Similarity Search in Vector DB
      ↓
Retrieve Top-K Chunks
      ↓
Build Prompt
      ↓
Send Context + Query to LLM
      ↓
LLM Generates Answer
      ↓
Return Answer to User
```

---

# 📦 Tech Stack

| Layer           | Technology             |
| --------------- | ---------------------- |
| Frontend        | HTML / React / Next.js |
| Backend         | Node.js + Express      |
| AI Framework    | LangChain              |
| LLM             | OpenAI GPT-4o          |
| Embedding Model | text-embedding-3-small |
| Vector Database | Pinecone               |
| Scraping        | Cheerio / Puppeteer    |
| Environment     | dotenv                 |

---

# 📁 Project Structure

```txt
rag-ecommerce-chatbot/
│
├── server/
│   ├── app.js
│   ├── ingest.js
│   ├── services/
│   │   ├── openai.js
│   │   ├── pinecone.js
│   │   └── rag.js
│   │
│   ├── routes/
│   │   └── chat.js
│   │
│   ├── data/
│   │   └── products.json
│   │
│   └── utils/
│       └── chunkText.js
│
├── client/
│   ├── index.html
│   ├── style.css
│   └── app.js
│
├── .env
├── package.json
└── README.md
```

---

# 🚀 Step 1 — Initialize Project

## Install Dependencies

```bash
npm init -y
```

```bash
npm install express cors dotenv openai langchain @langchain/openai @langchain/community @pinecone-database/pinecone
```

---

# 🔐 Step 2 — Environment Variables

Create `.env`

```env
OPENAI_API_KEY=your_openai_api_key
PINECONE_API_KEY=your_pinecone_api_key
PINECONE_INDEX=rag-ecommerce
PORT=5000
```

---

# 📄 Step 3 — Sample Knowledge Base

Create:

```txt
server/data/products.json
```

```json
[
  {
    "id": 1,
    "title": "Nike Air Max",
    "price": 4999,
    "description": "Lightweight running shoes with air cushioning.",
    "category": "Shoes"
  },
  {
    "id": 2,
    "title": "Gaming Laptop RTX 4060",
    "price": 89999,
    "description": "High performance gaming laptop with RTX graphics.",
    "category": "Laptop"
  }
]
```

---

# ✂️ Step 4 — Text Chunking Utility

Create:

```txt
server/utils/chunkText.js
```

```js
function chunkText(text, chunkSize = 300) {
  const chunks = [];

  for (let i = 0; i < text.length; i += chunkSize) {
    chunks.push(text.slice(i, i + chunkSize));
  }

  return chunks;
}

module.exports = chunkText;
```

---

# 🤖 Step 5 — OpenAI Configuration

Create:

```txt
server/services/openai.js
```

```js
const { OpenAIEmbeddings, ChatOpenAI } = require("@langchain/openai");

const embeddings = new OpenAIEmbeddings({
  model: "text-embedding-3-small",
  apiKey: process.env.OPENAI_API_KEY,
});

const llm = new ChatOpenAI({
  model: "gpt-4o-mini",
  apiKey: process.env.OPENAI_API_KEY,
  temperature: 0.3,
});

module.exports = {
  embeddings,
  llm,
};
```

---

# 🧠 Step 6 — Pinecone Configuration

Create:

```txt
server/services/pinecone.js
```

```js
const { Pinecone } = require("@pinecone-database/pinecone");

const pinecone = new Pinecone({
  apiKey: process.env.PINECONE_API_KEY,
});

const index = pinecone.index(process.env.PINECONE_INDEX);

module.exports = index;
```

---

# 📥 Step 7 — Ingest Data into Vector DB

Create:

```txt
server/ingest.js
```

```js
require("dotenv").config();

const fs = require("fs");
const path = require("path");

const chunkText = require("./utils/chunkText");
const index = require("./services/pinecone");
const { embeddings } = require("./services/openai");

async function ingest() {
  const filePath = path.join(__dirname, "data/products.json");

  const products = JSON.parse(fs.readFileSync(filePath, "utf8"));

  for (const product of products) {
    const text = `
      Product: ${product.title}
      Price: ₹${product.price}
      Description: ${product.description}
      Category: ${product.category}
    `;

    const chunks = chunkText(text);

    for (let i = 0; i < chunks.length; i++) {
      const chunk = chunks[i];

      const vector = await embeddings.embedQuery(chunk);

      await index.upsert([
        {
          id: `${product.id}-${i}`,
          values: vector,
          metadata: {
            text: chunk,
            title: product.title,
            price: product.price,
            category: product.category,
          },
        },
      ]);

      console.log(`Inserted chunk ${i}`);
    }
  }
}

ingest();
```

---

# 🔍 Step 8 — RAG Logic

Create:

```txt
server/services/rag.js
```

```js
const index = require("./pinecone");
const { embeddings, llm } = require("./openai");

async function askQuestion(question) {
  // Convert query into embedding
  const queryVector = await embeddings.embedQuery(question);

  // Similarity search
  const searchResult = await index.query({
    vector: queryVector,
    topK: 3,
    includeMetadata: true,
  });

  // Get chunks
  const context = searchResult.matches
    .map((match) => match.metadata.text)
    .join("\n\n");

  // Build prompt
  const prompt = `
You are an e-commerce AI assistant.

Answer the question only using the provided context.

Context:
${context}

Question:
${question}
  `;

  // Generate answer
  const response = await llm.invoke(prompt);

  return {
    answer: response.content,
    sources: searchResult.matches,
  };
}

module.exports = askQuestion;
```

---

# 🌐 Step 9 — Chat API Route

Create:

```txt
server/routes/chat.js
```

```js
const express = require("express");
const router = express.Router();

const askQuestion = require("../services/rag");

router.post("/chat", async (req, res) => {
  try {
    const { question } = req.body;

    const result = await askQuestion(question);

    res.json(result);
  } catch (error) {
    console.error(error);

    res.status(500).json({
      error: "Something went wrong",
    });
  }
});

module.exports = router;
```

---

# ⚙️ Step 10 — Express Server

Create:

```txt
server/app.js
```

```js
require("dotenv").config();

const express = require("express");
const cors = require("cors");

const chatRoutes = require("./routes/chat");

const app = express();

app.use(cors());
app.use(express.json());

app.use("/api", chatRoutes);

const PORT = process.env.PORT || 5000;

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

---

# 💻 Step 11 — Frontend Chat UI

Create:

```txt
client/index.html
```

```html
<!DOCTYPE html>
<html>
<head>
  <title>AI Chatbot</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <div class="chat-container">
    <h1>E-Commerce AI Assistant</h1>

    <div id="messages"></div>

    <input type="text" id="question" placeholder="Ask something...">

    <button onclick="sendMessage()">Send</button>
  </div>

  <script src="app.js"></script>
</body>
</html>
```

---

# 🎨 Step 12 — Frontend CSS

Create:

```txt
client/style.css
```

```css
body {
  font-family: Arial;
  background: #f5f5f5;
}

.chat-container {
  width: 500px;
  margin: 40px auto;
  background: white;
  padding: 20px;
  border-radius: 10px;
}

#messages {
  height: 300px;
  overflow-y: auto;
  border: 1px solid #ddd;
  padding: 10px;
  margin-bottom: 10px;
}

input {
  width: 80%;
  padding: 10px;
}

button {
  padding: 10px;
}
```

---

# 🧩 Step 13 — Frontend JavaScript

Create:

```txt
client/app.js
```

```js
async function sendMessage() {
  const input = document.getElementById("question");
  const messages = document.getElementById("messages");

  const question = input.value;

  messages.innerHTML += `<p><b>You:</b> ${question}</p>`;

  const response = await fetch("http://localhost:5000/api/chat", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ question }),
  });

  const data = await response.json();

  messages.innerHTML += `
    <p>
      <b>Bot:</b> ${data.answer}
    </p>
  `;

  input.value = "";
}
```

---

# ▶️ Step 14 — Run Application

## Start Backend

```bash
node server/app.js
```

## Ingest Data

```bash
node server/ingest.js
```

## Open Frontend

```txt
Open client/index.html in browser
```

---

# 🧠 Example Questions

```txt
Best shoes under ₹5000?
```

```txt
Suggest gaming laptop.
```

```txt
What is return policy?
```

---

# 🔥 Advanced Features

You can also add:

* Memory / Chat History
* Product Recommendation Engine
* Hybrid Search
* Redis Cache
* Multi-language Support
* Voice AI
* Admin Dashboard
* Re-ranking
* Streaming Responses
* Authentication
* WhatsApp Integration
* Shopify Integration

---

# 📈 Production Improvements

| Feature     | Recommended         |
| ----------- | ------------------- |
| Queue       | BullMQ              |
| Cache       | Redis               |
| Logging     | Winston             |
| Monitoring  | Grafana             |
| Deployment  | Docker + Kubernetes |
| API Gateway | Nginx               |
| CI/CD       | GitHub Actions      |

---

# 🏗️ Future Architecture

```txt
Frontend
   ↓
API Gateway
   ↓
Auth Service
   ↓
RAG Service
   ↓
Vector Database
   ↓
OpenAI API
```

---

# 📚 What You Learned

✅ RAG Architecture
✅ Embeddings
✅ Vector Databases
✅ Similarity Search
✅ LangChain
✅ OpenAI Integration
✅ Chunking
✅ Retrieval
✅ Prompt Engineering
✅ AI Chatbot Development

---

# ⭐ GitHub Description

Production-ready RAG E-Commerce AI Chatbot using LangChain, OpenAI, Pinecone, and Node.js.

---

# 🏷️ GitHub Topics

```txt
rag
langchain
openai
nodejs
ai-chatbot
vector-database
pinecone
llm
retrieval-augmented-generation
javascript
```

---

