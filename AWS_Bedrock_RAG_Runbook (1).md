# 🚀 Building a RAG System on AWS
### Amazon Bedrock Knowledge Bases + Amazon Lex | Step-by-Step Guide
> **Level:** Beginner-Friendly | **Platform:** AWS Console Only | **No Coding Required**

---

## 📋 Table of Contents

**Part A — Amazon Bedrock Knowledge Base**
1. [What is RAG?](#1-what-is-rag)
2. [AWS Architecture Overview](#2-aws-architecture-overview)
3. [Prerequisites & Setup](#3-prerequisites--setup)
4. [Step 1 — Enable Model Access](#4-step-1--enable-model-access)
5. [Step 2 — Create an S3 Bucket & Upload Documents](#5-step-2--create-an-s3-bucket--upload-documents)
6. [Step 3 — Create the Knowledge Base](#6-step-3--create-the-knowledge-base)
7. [Step 4 — Sync the Data Source](#7-step-4--sync-the-data-source)
8. [Step 5 — Test in the AWS Console](#8-step-5--test-in-the-aws-console)

**Part B — Amazon Lex Chatbot with RAG**

9. [What is Amazon Lex?](#9-what-is-amazon-lex)
10. [Step 6 — Create an Amazon Lex Bot](#10-step-6--create-an-amazon-lex-bot)
11. [Step 7 — Add QnAIntent to the Bot](#11-step-7--add-qnaintent-to-the-bot)
12. [Step 8 — Connect Lex to Your Knowledge Base](#12-step-8--connect-lex-to-your-knowledge-base)
13. [Step 9 — Build & Test the Lex Chatbot](#13-step-9--build--test-the-lex-chatbot)

**Reference**

14. [Key Concepts Cheat Sheet](#14-key-concepts-cheat-sheet)
15. [Troubleshooting Guide](#15-troubleshooting-guide)
16. [Cleanup — Delete Resources](#16-cleanup--delete-resources)

---

# PART A — Amazon Bedrock Knowledge Base

---

## 1. What is RAG?

### The Problem RAG Solves

LLMs like Claude or GPT are trained on data up to a certain cutoff date. They **don't know**:
- Your company's internal documents
- Proprietary policies, manuals, or product specs
- Events after their training cutoff

### How RAG Works (Simple Analogy)
> Think of it like giving an open-book exam to a brilliant student (the LLM). Without RAG, they answer from memory. **With RAG**, they can look up answers in your specific textbook before answering.

### The RAG Pipeline (Step by Step)

```
Your Documents (PDF, TXT, DOCX)
        ↓
  [Chunking] — Split into smaller pieces
        ↓
  [Embedding] — Convert text into numbers (vectors)
        ↓
  [Vector Store] — Store vectors in a searchable database
        ↓
  User asks a question
        ↓
  Question → converted to vector → searched against stored vectors
        ↓
  Top matching chunks retrieved
        ↓
  LLM gets: Question + Retrieved Chunks → Generates Answer
```

### AWS Services Used in This Guide

| Service | Role |
|---|---|
| **Amazon S3** | Stores your source documents |
| **Amazon Bedrock** | Hosts the Foundation Models (LLMs) |
| **Bedrock Knowledge Bases** | Manages the full RAG pipeline |
| **Amazon OpenSearch Serverless** | The vector database (auto-created) |
| **Amazon Titan** | Embedding model (converts text → vectors) |
| **Claude (Anthropic)** | The LLM that generates final answers |
| **Amazon Lex** | Conversational chatbot interface on top of RAG |

---

## 2. AWS Architecture Overview

### Part A — Bedrock Knowledge Base
```
┌─────────────────────────────────────────────────────────┐
│              Amazon Bedrock (RetrieveAndGenerate)        │
│                                                          │
│   ┌──────────────┐    ┌────────────────────────────┐    │
│   │  Knowledge   │◄──►│  OpenSearch Serverless     │    │
│   │    Base      │    │  (Vector Store)            │    │
│   └──────┬───────┘    └────────────────────────────┘    │
│          │                                               │
│   ┌──────▼───────┐    ┌────────────────────────────┐    │
│   │  Titan       │    │  Claude (LLM)              │    │
│   │  Embeddings  │    │  Generates Final Answer    │    │
│   └──────────────┘    └────────────────────────────┘    │
└───────────────────────────┬─────────────────────────────┘
                            │ Syncs from
                            ▼
                   ┌─────────────────┐
                   │   Amazon S3     │
                   │  (Your Docs)   │
                   └─────────────────┘
```

### Part B — Amazon Lex + Knowledge Base (Full RAG Chatbot)
```
  User types a question
        ↓
  Amazon Lex (QnAIntent)
        ↓
  Bedrock Knowledge Base
        ↓  Titan converts question → vector
        ↓  OpenSearch finds closest document chunks
        ↓  Claude generates answer from chunks
        ↓
  Lex sends answer back to user 💬
```

---

## 3. Prerequisites & Setup

### 🔑 What You Need Before Starting
- [ ] An **AWS Account** (Free Tier works for this demo)
- [ ] **AWS Region**: Use `us-east-1` (N. Virginia) — best model availability
- [ ] A few **sample documents** to upload (PDF or TXT files)

### 📄 Sample Documents to Use

Use documents about a topic the LLM *wouldn't already know*:
- Your college/company **employee handbook** (PDF)
- A **product manual** (PDF)
- Any **custom FAQ document** you create as a .txt file

**Quick Option — Create a demo text file right now:**
```
Save this as company_faq.txt:

---
Q: What is the return policy?
A: Products can be returned within 30 days of purchase with original receipt.

Q: What are the office hours?
A: Our office is open Monday to Friday, 9am to 6pm EST.

Q: Who is the CEO?
A: Jane Smith has been the CEO since 2021.

Q: What is our vacation policy?
A: Full-time employees receive 15 days of paid vacation per year.
---
```

---

## 4. Step 1 — Enable Model Access

> ⚠️ This is **required** before anything else works.

### 1.1 — Go to Amazon Bedrock
1. Log in to [AWS Console](https://console.aws.amazon.com)
2. Search for **"Bedrock"** in the top search bar
3. Click **Amazon Bedrock**
4. Make sure you are in **us-east-1** (check top-right corner)

### 1.2 — Request Model Access
1. In the left sidebar, scroll down and click **"Model access"**
2. Click the orange **"Manage model access"** button
3. Find and check these two models:
   - ✅ **Amazon Titan Text Embeddings V2** (for creating vectors)
   - ✅ **Anthropic Claude 3 Sonnet** (for generating answers)
4. Click **"Save changes"** at the bottom
5. Wait for status to show **"Access granted"** ✅ (usually instant)

> 💡 **Tip:** If you get an error on a brand-new account, try launching a small EC2 instance and terminating it — this activates the account for Bedrock access.

---

## 5. Step 2 — Create an S3 Bucket & Upload Documents

### 2.1 — Create an S3 Bucket
1. Search for **"S3"** in the AWS Console
2. Click **"Create bucket"**
3. Fill in:
   - **Bucket name:** `rag-demo-docs-[your-initials]-[random-number]` *(must be globally unique)*
     - Example: `rag-demo-docs-jd-4821`
   - **Region:** `us-east-1`
   - Leave everything else as default
4. Click **"Create bucket"** ✅

### 2.2 — Upload Your Documents
1. Click on your newly created bucket
2. Click **"Upload"**
3. Click **"Add files"**
4. Select your PDF / TXT documents
5. Click **"Upload"** ✅

> 📌 **Supported file types:** `.pdf`, `.txt`, `.md`, `.html`, `.doc/.docx`, `.csv`, `.xls/.xlsx`

---

## 6. Step 3 — Create the Knowledge Base

> ⚠️ After clicking Create, AWS will provision an OpenSearch Serverless collection. This can take 3–5 minutes — that's normal!

### 3.1 — Navigate to Knowledge Bases
1. Go back to **Amazon Bedrock**
2. In the left sidebar, under **"Builder tools"**, click **"Knowledge bases"**
3. Click **"Create knowledge base"**

### 3.2 — Knowledge Base Details
1. **Name:** `student-rag-demo`
2. **Description:** `RAG demo for student workshop`
3. Under **IAM Role**: Select **"Create and use a new service role"** (auto-creates permissions)
4. Click **"Next"**

### 3.3 — Configure Data Source
1. Select **"Amazon S3"** as your data source
2. **Data source name:** `my-documents-source`
3. Under **S3 URI**: Click **"Browse S3"** and select the bucket you created in Step 2
4. Click **"Next"**

### 3.4 — Choose Embedding Model
1. Under **Embeddings model**, click **"Choose model"**
2. Select: **Titan Text Embeddings V2** (by Amazon)
3. Click **"Apply"**

### 3.5 — Configure Vector Store
1. Select **"Quick create a new vector store"**
2. This auto-creates an **Amazon OpenSearch Serverless** collection for you
3. Click **"Next"**

### 3.6 — Review & Create
1. Review all settings
2. Click **"Create knowledge base"**
3. ⏳ Wait for the green **"Knowledge base created successfully"** banner ✅

> 📌 **Important:** Copy and save your **Knowledge Base ID** from the details page — you will need it in Part B when connecting Lex.

---

## 7. Step 4 — Sync the Data Source

> Creating the knowledge base is just the setup. **Syncing** is what actually processes your documents — it chunks them, embeds them, and stores them in the vector database.

### 4.1 — Start the Sync
1. Inside your Knowledge Base page, click on the **"Data source"** tab
2. Select your data source (`my-documents-source`)
3. Click **"Sync"**
4. ⏳ Wait for status to show **"Sync complete"** ✅

### What Happens During Sync
```
Your PDFs in S3
    ↓  Bedrock reads each file
    ↓  Splits into chunks (~300-500 tokens each)
    ↓  Calls Titan Embeddings to convert each chunk to a vector
    ↓  Stores vectors in OpenSearch Serverless
    ✅  Knowledge base is now searchable!
```

---

## 8. Step 5 — Test in the AWS Console

### 5.1 — Open the Test Console
1. On your Knowledge Base page, click **"Test knowledge base"** (top-right button)
2. A chat panel appears on the right side of the screen

### 5.2 — Choose Your Model
1. Click **"Select model"** in the test panel
2. Choose **Anthropic → Claude 3 Sonnet**
3. Click **"Apply"**

### 5.3 — Ask Questions!

```
If you uploaded the company FAQ file:
→ "What is the return policy?"
→ "What are the office hours?"
→ "Who is the CEO of the company?"
→ "How many vacation days do employees get?"
```

```
If you uploaded a product manual:
→ "How do I reset the device?"
→ "What are the warranty terms?"
→ "What is included in the box?"
```

### 5.4 — Observe Source Citations
- Notice the **"Show source"** links in each response
- Click them to see exactly which chunk from your document was retrieved
- This is called **grounding** — the model shows its work!

---

# PART B — Amazon Lex RAG Chatbot

---

## 9. What is Amazon Lex?

Amazon Lex is AWS's service for building **conversational chatbots** — the same technology that powers Alexa. In this section, you will turn your Bedrock Knowledge Base into a **fully conversational chatbot** using Lex.

### How Lex + Bedrock Knowledge Base Works Together

The key feature that makes this possible is called **QnAIntent** — a built-in Lex intent that directly connects to a Bedrock Knowledge Base. You don't need to define every possible question. Instead:

```
User asks anything → Lex routes it to QnAIntent
                            ↓
              QnAIntent queries your Knowledge Base
                            ↓
              Claude generates a grounded answer
                            ↓
              Lex sends the answer back to the user 💬
```

### Why Use Lex Instead of Just Bedrock?

| Bedrock Knowledge Base (Part A) | + Amazon Lex (Part B) |
|---|---|
| Raw RAG query via console | Full chatbot with conversation flow |
| No user interface | Built-in chat UI you can embed anywhere |
| No voice support | Supports text AND voice |
| API / developer tool | Ready-to-use for end users |

---

## 10. Step 6 — Create an Amazon Lex Bot

### 6.1 — Navigate to Amazon Lex
1. In the AWS Console, search for **"Lex"**
2. Click **Amazon Lex**
3. Make sure you are still in **us-east-1**
4. Click **"Create bot"**

### 6.2 — Bot Creation Method
1. Under **Creation method**, select **"Create a blank bot"**
2. Click **"Next"**

### 6.3 — Bot Configuration
Fill in the following:

| Field | Value |
|---|---|
| **Bot name** | `RAGDemoBot` |
| **Description** | `Student demo chatbot powered by Bedrock KB` |
| **IAM permissions** | Select **"Create a role with basic Amazon Lex permissions"** |
| **COPPA** | Select **"No"** |

4. Click **"Next"**

### 6.4 — Language Settings
1. **Language:** English (US)
2. **Voice interaction:** Select any voice (or "None" for text-only)
3. **Intent classification confidence score threshold:** Leave as default (`0.40`)
4. Click **"Done"**

> ✅ Your bot is now created. You will land on the **Intents** page inside the bot editor.

---

## 11. Step 7 — Add QnAIntent to the Bot

**QnAIntent** is the built-in Lex feature that connects directly to your Bedrock Knowledge Base. It handles any free-form question from users automatically — no manual intent configuration needed.

### 7.1 — Add a New Intent
1. In the left sidebar of the bot editor, click **"Intents"**
2. Click **"Add intent"**
3. Select **"Use built-in intent"**
4. From the dropdown, search for and select **`AMAZON.QnAIntent`**
5. Click **"Add"**

### 7.2 — You Are Now on the QnAIntent Configuration Page
- You will see a section called **"QnA source"**
- Proceed directly to Step 8 to connect your Knowledge Base here

---

## 12. Step 8 — Connect Lex to Your Knowledge Base

### 8.1 — Select the Knowledge Base Source
1. On the QnAIntent page, under **"QnA source"**, select **"Bedrock knowledge base"**

### 8.2 — Pick Your Knowledge Base
1. A dropdown appears — select the Knowledge Base you created in Part A (`student-rag-demo`)

> 💡 If your Knowledge Base doesn't appear, make sure both Lex and the Knowledge Base are in the same region (`us-east-1`).

### 8.3 — Select the Response Generation Model
1. Under **"Model to generate responses"**, click the dropdown
2. Select **Anthropic → Claude 3 Sonnet**
3. This is the LLM that reads the retrieved chunks and writes the final answer

### 8.4 — Save the Intent
1. Scroll to the bottom and click **"Save intent"** ✅

### 8.5 — Configure the Fallback Intent
For the best experience, make sure Lex gracefully handles unanswered questions:

1. In the left sidebar, click **"Fallback intent"** (auto-created by Lex)
2. Under **"Closing response"**, set the message to:
   `"I'm sorry, I couldn't find that in the knowledge base. Please try rephrasing your question."`
3. Click **"Save intent"** ✅

---

## 13. Step 9 — Build & Test the Lex Chatbot

### 9.1 — Build the Bot
1. In the top-right of the bot editor, click the orange **"Build"** button
2. ⏳ Wait for the build to complete (usually 1–2 minutes)
3. You will see a **"Build successful"** confirmation ✅

### 9.2 — Test the Bot in the Console
1. After the build completes, click **"Test"** (top-right)
2. A chat window opens on the right side
3. Type your questions and press Enter

### 9.3 — Try These Test Questions
```
→ "What is the return policy?"
→ "What are the office hours?"
→ "Who is the CEO?"
→ "How many vacation days do employees receive?"
```

### 9.4 — What a Good Response Looks Like
- The bot returns a **specific, accurate answer** pulled from your documents
- Responses are grounded in the content you uploaded to S3
- If a question is outside your documents, the bot responds with the fallback message — that is correct behavior!

### 9.5 — Compare: Bedrock Console vs Lex Chatbot

| Feature | Bedrock Test Console | Amazon Lex Chatbot |
|---|---|---|
| Interface | Developer console | Conversational chat UI |
| Multi-turn memory | No | Yes (session-based) |
| Source citations | Visible | Hidden (answer only) |
| Voice support | No | Yes |
| Embeddable in apps | No | Yes |
| Best for | Testing & debugging | End-user demos |

### 9.6 — Create a Bot Alias (Required for Sharing)

If you want to share the bot or connect it to a web UI:

1. In the left sidebar, click **"Aliases"**
2. Click **"Create alias"**
3. **Alias name:** `DemoAlias`
4. Under **"Bot version"**, select the version you just built (or `Draft`)
5. Click **"Create"** ✅

> 📌 An alias is a stable pointer to a specific bot version — it allows updates without breaking any links.

---

# Reference

---

## 14. Key Concepts Cheat Sheet

```
┌─────────────────────────────────────────────────────────────┐
│                    RAG KEY CONCEPTS                          │
├────────────────────┬────────────────────────────────────────┤
│ Term               │ Simple Explanation                      │
├────────────────────┼────────────────────────────────────────┤
│ RAG                │ Give the LLM your documents to look up  │
│ Embedding          │ Converting text to numbers (vectors)    │
│ Vector Store       │ Database of those number-representations│
│ Chunking           │ Splitting documents into searchable bits │
│ Semantic Search    │ Search by MEANING, not exact keywords   │
│ Knowledge Base     │ AWS-managed RAG pipeline                │
│ Foundation Model   │ Pre-trained LLM (Claude, Titan, etc.)  │
│ Retrieval          │ Finding the most relevant chunks        │
│ Generation         │ LLM writes the final answer             │
│ Citation           │ Source document the answer came from    │
│ Sync               │ Processing & indexing new documents     │
│ OpenSearch         │ The vector database AWS uses by default │
│ Titan Embeddings   │ Amazon's model for creating vectors     │
│ Amazon Lex         │ AWS chatbot service (voice + text)      │
│ QnAIntent          │ Built-in Lex intent that queries KB     │
│ Bot Alias          │ A named pointer to a specific bot build │
└────────────────────┴────────────────────────────────────────┘
```

## 16. Cleanup — Delete Resources

> ⚠️ **Always clean up after your demo to avoid unexpected AWS charges!**

### Cleanup Order (follow this exact sequence)

1. **Delete the Lex Bot**
   - Go to Amazon Lex → Bots
   - Select `RAGDemoBot` → Click **"Delete"**

2. **Delete the Knowledge Base**
   - Go to Bedrock → Knowledge Bases
   - Select your KB → Click **"Delete"**
   - This also removes the associated OpenSearch Serverless collection

3. **Delete the S3 Bucket**
   - Go to S3 → Select your bucket
   - First **empty** the bucket (Actions → Empty)
   - Then **delete** the bucket

4. **Verify OpenSearch is deleted**
   - Go to **Amazon OpenSearch Service → Serverless → Collections**
   - Confirm no collections remain (delete manually if needed)

---
