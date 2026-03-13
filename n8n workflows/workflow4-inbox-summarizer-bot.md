# Workflow 4: AI Inbox Summarizer Bot – Progress Report

This bot acts as an **AI-powered inbox summarizer**. It connects to one or more message sources (email-like feeds, ticketing exports, or daily logs from tools like Nano Banana / AI Studio / Flow) and produces:

1. A **daily executive summary** of important items
2. A **structured list of follow-ups** (with owners and due dates)
3. An optional **Telegram notification** with the top 3–5 highlights

Unlike Workflows 1–3, this bot is **standalone**: it does not call or depend on the other workflows. It is designed so an operator can run it independently to “get the day’s picture” at a glance.

## Executive Summary
- **Purpose**: Turn a noisy inbox (log of messages/requests) into a clear daily briefing with prioritized follow-ups.
- **AI Intelligence**: Groups related messages, identifies important themes, and extracts action items with suggested owners and due dates.
- **Automation**: Can be scheduled to run once per day (e.g., via n8n Cron) and send a summary plus a structured log to a sheet/DB.

---

## Node Setup & Configuration

### 1. Trigger – Manual or Scheduled

You can run this either manually or on a schedule.

**Option A: Manual (Webhook Trigger)**
- **Node**: Webhook
- **Method**: `POST`
- **Path**: `inbox-summary`
- **Production URL**: `http://localhost:5678/webhook/inbox-summary`
- **Expected JSON Body**:
  ```json
  {
    "source": "nano-banana | aistudio | flow | export",
    "messages": [
      {
        "id": "1",
        "from": "user@example.com",
        "subject": "optional subject",
        "body": "full message text",
        "timestamp": "2026-03-12T03:00:00Z"
      }
    ],
    "requester": "who is running this (e.g., operator)"
  }
  ```
- **Respond**: `Using 'Respond to Webhook' Node`

**Option B: Scheduled (Cron + Source Node)**
- **Node 1**: `Cron` – runs e.g. every weekday at 09:00.
- **Node 2**: Source integration
  - Example: HTTP / Database / Stitch / Google Sheets / Notion.
  - This node pulls the last 24h of messages into an array.

Both options feed into a **normalization Code node** that produces a common `messages` array.

### 2. Code Node – Normalize Inbox Items

- **Mode**: `Run Once for All Items`
- **Language**: JavaScript
- **Goal**: Ensure we have a clean `messages` array no matter where data came from.

```javascript
return $input.all().map(item => {
  const j = item.json;

  // If messages already present (webhook path), use them; otherwise expect items[] from previous node
  const messages = Array.isArray(j.messages) ? j.messages : ($input.all().map(i => i.json));

  const normalized = messages.map((m, index) => ({
    id: m.id || String(index + 1),
    from: m.from || m.sender || "unknown",
    subject: m.subject || m.title || "(no subject)",
    body: m.body || m.text || "",
    timestamp: m.timestamp || m.date || null
  }));

  return {
    json: {
      source: j.source || "mixed",
      requester: j.requester || "unknown",
      messages: normalized
    }
  };
});
```

### 3. AI Model Node – Daily Summary & Actions

Use Gemini / OpenAI / Flow with strict JSON output.

- **Temperature**: `0.2`
- **Prompt**:
  ```text
  You are an AI assistant creating a daily executive inbox summary from a batch of messages.

  Return ONLY a valid minified JSON object with the following keys:
  date (string, ISO date for this summary)
  overall_summary (string, max 200 words)
  highlights (array of 3-10 short strings describing the most important items)
  action_items (array of objects with keys: description, owner_suggestion, due_date_iso, priority)
  risks (array of short strings, can be empty)

  Rules:
  - Never output markdown.
  - Never output explanation outside the JSON.
  - Use date = today's date in ISO (YYYY-MM-DD) if you are unsure.
  - priority must be one of: low, medium, high.
  - If there are no clear risks, return risks as an empty array.

  Here is the input inbox in JSON form:
  {{$json.messages}}
  ```

### 4. Code Node – Validate & Normalize AI Output

- **Mode**: `Run Once for All Items`
- **Language**: JavaScript

```javascript
return $input.all().map(item => {
  function findJsonString(obj) {
    if (typeof obj === 'string' && obj.includes('{')) return obj;
    if (obj && typeof obj === 'object') {
      for (const k in obj) {
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
      date: new Date().toISOString().slice(0, 10),
      overall_summary: "Failed to parse AI output.",
      highlights: [],
      action_items: [],
      risks: []
    };
  }

  const pick = (v, allowed, fallback) => allowed.includes(v) ? v : fallback;

  const normalized = {
    date: String(d.date || new Date().toISOString().slice(0, 10)),
    overall_summary: String(d.overall_summary || "No summary provided."),
    highlights: Array.isArray(d.highlights) ? d.highlights.map(String) : [],
    action_items: Array.isArray(d.action_items) ? d.action_items.map(a => ({
      description: String(a.description || ""),
      owner_suggestion: String(a.owner_suggestion || "unassigned"),
      due_date_iso: a.due_date_iso || null,
      priority: pick(a.priority || "medium", ["low","medium","high"], "medium")
    })) : [],
    risks: Array.isArray(d.risks) ? d.risks.map(String) : []
  };

  return { json: normalized };
});
```

### 5. Action Nodes – Logging & Notification

#### A. Log to Google Sheets / Notion / DB

Log one row per daily summary (not per message):

- **Columns**:
  - `date`
  - `overall_summary`
  - `highlights (joined as text)`
  - `action_items (stringified JSON)`
  - `risks (joined as text)`

This gives you a historical record you can later connect to **Stitch** or another analytics tool.

#### B. Telegram Notification (Optional)

Send a short, operator-friendly message:

```text
 Daily Inbox Summary ({{$json.date}})

{{$json.overall_summary}}

Top highlights:
- {{$json.highlights[0]}}
- {{$json.highlights[1]}}
- {{$json.highlights[2]}}
```

*(Guard against missing indices by either limiting to available items or using expressions with defaults.)*

### 6. Respond to Webhook (If Using Webhook Trigger)

If you used the Webhook trigger, connect all branches to a `Respond to Webhook` node.

- **Respond With**: Manual JSON
  - Example:
    ```json
    {
      "date": "{{$json.date}}",
      "overall_summary": "{{$json.overall_summary}}",
      "highlights": {{$json.highlights}},
      "action_items": {{$json.action_items}}
    }
    ```

---

## How This Fits the Overall System
- **AI Automation / AI Agent**: This is an autonomous summarization agent that turns raw message feeds into daily briefings with concrete follow-ups.
- **Independent Chatbot/Automation**: It is **separate from Workflows 1–3** – no shared webhooks or routing – so it can be deployed and evaluated on its own.
- **Tool Practice (Nano Banana, AI Studio, Flow, Stitch)**:
  - Message exports or data streams from these tools feed into the bot as the `messages` array.
  - The AI model can be Gemini/OpenAI/Flow.
  - The output summary log can be wired to Stitch or a warehouse for long-term analytics.
