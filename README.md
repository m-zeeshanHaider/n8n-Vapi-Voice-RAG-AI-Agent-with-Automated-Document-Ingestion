# Vapi Supabase Voice RAG Agent

An n8n workflow that answers voice assistant questions from a Supabase vector database. It receives a Vapi tool call over a webhook and returns a one-sentence answer with a source citation.

![Illustration of a friendly headset robot beside the n8n and Vapi logos, a document with a magnifying glass, and a microphone](images/thumbnail.png)

## Overview

The system gives a Vapi voice assistant a knowledge base to query. Vapi sends the caller's question to an n8n webhook. An AI agent searches a Supabase vector store of embedded documents, reranks the matches, and writes a one-sentence answer from them. The answer goes back to Vapi in the tool call response format.

![Canvas view of the workflow: a Webhook node feeds an AI Agent node, which has a Gemini chat model, Postgres chat memory, and a Supabase vector database tool with Gemini embeddings attached, followed by an Edit Fields node and a Respond to Webhook node](images/Vapi-Voice-RAG-AI-Agent-with-Automated-Document-Ingestion-1.PNG)

## Workflows

| Workflow | Trigger | Purpose |
| --- | --- | --- |
| Vapi Supabase Voice RAG Agent | Webhook (POST) | Answers a Vapi tool call question from the Supabase knowledge base and returns the answer to Vapi. |

## Workflow Details

### Vapi Supabase Voice RAG Agent

This workflow turns a Vapi tool call into a cited, one-sentence answer retrieved from a vector database.

**Trigger:** A Webhook node accepts POST requests and holds the response until a Respond to Webhook node replies.

Execution steps:

1. The Webhook node receives the Vapi request, which contains a tool call.
2. The AI Agent node takes the `query` argument of the first tool call as its prompt.
3. The agent loads the last 10 messages from Postgres Chat Memory.
4. The agent calls the Supabase Vector Database tool, which embeds the query with Google Gemini embeddings and retrieves the 10 closest document chunks.
5. A Cohere reranker reorders the retrieved chunks by relevance.
6. A Google Gemini 2.5 Flash chat model writes the answer. The agent falls back to Gemini 2.5 Flash Lite if the primary model fails.
7. The Set node removes double quote characters from the agent output.
8. The Respond to Webhook node returns a JSON body with the tool call ID from the request and the answer as the result.

Agent behavior:

- The system message limits every answer to one sentence followed by a source citation.
- The agent answers only from retrieved documents.
- The agent replies "No relevant information found in the knowledge base." when retrieval returns nothing.
- The memory session key combines the workflow and execution identifiers, so each webhook call starts a new session.
- The AI Agent, Webhook, Set, and Respond to Webhook nodes retry on failure.

Branching and loops: the workflow has no branches, loops, or sub-workflow calls. The workflow settings reference an error workflow that is not part of this repository.

Integrations: Vapi, Google Gemini, Supabase, PostgreSQL, Cohere.

## Integrations

- Vapi sends the tool call and receives the answer.
- Google Gemini provides the chat models and the embeddings.
- Supabase stores the document vectors and serves the similarity search.
- PostgreSQL stores the chat memory.
- Cohere reranks the retrieved chunks.

## Prerequisites

- An n8n instance with the AI (LangChain) nodes available.
- A Vapi account with a tool configured to call the webhook.
- A Supabase project with a vector table of embedded documents and a matching search function.
- A PostgreSQL database that n8n can reach.
- A Google AI account for Gemini.
- A Cohere account.
- Credentials of these types in n8n: Supabase API, Postgres, Google Gemini (PaLM) API, and Cohere API.

## Setup

1. Import `workflows/Vapi_Supabase_Voice_RAG_Agent.json` into n8n.
2. Create a credential in n8n for each type listed under Prerequisites.
3. Assign the credentials to the Postgres Chat Memory, Supabase Vector Database, Embeddings Gemini, Reranker Cohere, and both Gemini chat model nodes.
4. Open the Supabase Vector Database node and select your vector table and search function.
5. In the workflow settings, select your own error workflow or clear the error workflow setting.
6. Activate the workflow.
7. Configure your Vapi tool to send POST requests to the production URL of the Webhook node.
