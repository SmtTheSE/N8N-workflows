# Workflow 3: AI Knowledge Agent – Progress Report

This bot receives long-form input (URLs, docs, or raw text), calls an AI model to generate structured summaries and action items, and then routes the result to both a Telegram channel and a long-term knowledge log (e.g., Google Sheets/Docs/Notion). It behaves like a lightweight research/meeting-note agent that standardizes insights for the team.

## Executive Summary
- **Purpose**: Turn messy inputs (links, notes, transcripts) into a clean, standardized summary with key decisions and todos.
- **AI Intelligence**: Extracts title/topic, context, 5 key points, risks, and concrete next actions. Rates importance and suggests an owner.
- **Automation**: For high-importance items, sends a condensed summary to Telegram; everything is logged silently to a knowledge base.

---

## Node Setup & Configuration

### 1. Webhook (Trigger)
- **Method**: `POST`
- **Path**: `knowledge-agent`
- **Production URL**: `http://localhost:5678/webhook/knowledge-agent`
- **Expected JSON Body**:
  ```json
  {
    "source": "nano-banana | notebooklm | aistudio | manual",
    "title": "optional short title",
    "url": "optional url",
    "raw_text": "full text or notes here",
    "requester": "who is sending this"
  }
  ```
- **Respond**: `Using 'Respond to Webhook' Node`

### 2. (Optional) HTTP Node – Fetch Content from URL
Use this if you want to automatically pull page text when a URL is present.

- **Node Name**: `Fetch URL Content`
- **HTTP Method**: `GET`
- **URL**: `{{$json["url"]}}`
- **Continue On Fail**: `true`

Combine the body with `raw_text` later in a Code Node.

### 3. Code Node – Build Unified Input Text
- **Mode**: `Run Once for All Items`
- **Language**: JavaScript
- **Code**:
  ```javascript
  return $input.all().map(item => {
    const j = item.json;
    const fetched = j["body"] || j["data"] || ""; // from HTTP node if connected

    const combinedText = [
      j.raw_text || "",
      fetched
    ].filter(Boolean).join("\n\n");

    return {
      json: {
        source: j.source || "manual",
        title: j.title || "Untitled Insight",
        url: j.url || null,
        requester: j.requester || "unknown",
        combined_text: combinedText
      }
    };
  });
  ```

### 4. AI Model Node (Gemini / OpenAI / Flowise, etc.)
Use the same strict JSON strategy as in the other workflows.

- **Temperature**: `0.2`
- **Prompt**:
  ```text
  You are an AI knowledge and research summarization agent.

  Return ONLY a valid minified JSON object with the following keys:
  topic (string)
  importance (one of: low, medium, high)
  summary (string, max 200 words)
  key_points (array of 3-7 short bullet strings)
  decisions (array of strings, can be empty)
  action_items (array of objects with keys: description, owner_suggestion, due_date_iso)
  risks (array of strings, can be empty)
  suggested_owner (string)

  Rules:
  - Never output markdown.
  - Never output explanation.
  - If unsure, set importance to "medium".
  - If no decisions/actions, return empty arrays.

  Context:
  Source: {{$json.source}}
  Title: {{$json.title}}
  URL: {{$json.url}}
  Requester: {{$json.requester}}

  Full Text:
  {{$json.combined_text}}
  ```

### 5. Code Node – Validate & Normalize AI Output
- **Mode**: `Run Once for All Items`
- **Code**:
  ```javascript
  return $input.all().map(item => {
    function findJsonString(obj) {
      if (typeof obj === 'string' && obj.includes('{')) return obj;
      if (obj && typeof obj === 'object') {
        for (let k in obj) {
          const found = findJsonString(obj[k]);
          if (found) return found;
        }
      }
      return null;
    }

    let raw = findJsonString(item.json) || "";
    raw = raw.replace(/```json/g, "").replace(/```/g, "").trim();

    let d;
    try {
      d = JSON.parse(raw);
    } catch (e) {
      d = {
        topic: item.json.title || "Untitled",
        importance: "medium",
        summary: "Failed to parse AI output.",
        key_points: [],
        decisions: [],
        action_items: [],
        risks: [],
        suggested_owner: "unassigned"
      };
    }

    const pick = (v, allowed, fallback) => allowed.includes(v) ? v : fallback;

    const normalized = {
      topic: String(d.topic || item.json.title || "Untitled"),
      importance: pick(d.importance || "medium", ["low","medium","high"], "medium"),
      summary: String(d.summary || "No summary provided."),
      key_points: Array.isArray(d.key_points) ? d.key_points.map(String) : [],
      decisions: Array.isArray(d.decisions) ? d.decisions.map(String) : [],
      action_items: Array.isArray(d.action_items) ? d.action_items.map(a => ({
        description: String(a.description || ""),
        owner_suggestion: String(a.owner_suggestion || d.suggested_owner || "unassigned"),
        due_date_iso: a.due_date_iso || null,
      })) : [],
      risks: Array.isArray(d.risks) ? d.risks.map(String) : [],
      suggested_owner: String(d.suggested_owner || "unassigned")
    };

    return { json: normalized };
  });
  ```

### 6. IF Node – High-Importance Routing
- **Condition (String)**:
  - `{{ $json.importance }}` **is equal to** `high`

### 7. Action Nodes

**True Branch (High Importance → Telegram)**
- **Telegram Node** (Send Message)
  - Template:
    ```text
     [{{$json.importance}}] {{$json.topic}}
    Summary: {{$json.summary}}

    Key points:
    - {{$json.key_points[0]}}
    - {{$json.key_points[1]}}
    - {{$json.key_points[2]}}

    Owner: {{$json.suggested_owner}}
    ```

**False Branch (Medium/Low Importance)**
- **Google Sheets / Notion / DB Node**:
  - Append row with fields:
    - `timestamp`
    - `topic`
    - `importance`
    - `summary`
    - `decisions (joined as text)`
    - `action_items (stringified JSON)`
    - `risks (joined as text)`

*(You can also run this node on both branches so all insights get logged, and only high-importance ones ping Telegram.)*

### 8. Respond to Webhook (End)
- Connect BOTH branches to this node.
- **Respond With**: `All Incoming Item Data`

---

## Testing (Production Mode)

With the workflow **Active**, run these examples:

**1. High-Importance Strategy Memo (Should Trigger Telegram)**
```bash
curl -X POST "http://localhost:5678/webhook/knowledge-agent" \
  -H "Content-Type: application/json" \
  -d '{
    "source": "notebooklm",
    "title": "Q3 Product Strategy",
    "raw_text": "Here are the final decisions for Q3: we are sunsetting legacy feature X, doubling down on AI automation, and reallocating 40% of engineering capacity.",
    "requester": "Executive"
  }'
```

**2. Low-Importance Research Link (Log Only)**
```bash
curl -X POST "http://localhost:5678/webhook/knowledge-agent" \
  -H "Content-Type: application/json" \
  -d '{
    "source": "nano-banana",
    "title": "Interesting AI article",
    "url": "https://example.com/ai-article",
    "raw_text": "", 
    "requester": "Intern"
  }'
```

---

## How This Fits the Practice Plan
- **AI Automation / Agent**: This workflow behaves like a knowledge agent that continuously turns inputs from tools like Nano Banana, NotebookLM, AI Studio, etc., into structured insights.
- **Chatbot / Escalation Pattern**: Reuses the same webhook + AI + code + IF + Telegram pattern from the other workflows, reinforcing the mental model.
- **Extensibility**: Later, you can:
  - Add a second webhook that accepts chat-style questions and returns answers from the logged knowledge base.
  - Integrate with Stitch or data warehouses for analytics.
