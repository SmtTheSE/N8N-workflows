# Workflow 1: AI Task Triage Bot – Progress Report

This bot receives unstructured feature requests, bugs, or general tasks, categorizes them using an AI model, and routes them appropriately. High-priority issues are immediately escalated to Telegram, while others are processed silently for backlog integration.

## Executive Summary
- **Purpose**: Automate the triage of incoming tasks, support requests, and internal messages.
- **AI Intelligence**: Classifies text by priority (urgent/high/normal/low), category (bug/feature/ops, etc.), and suggests owners and next actions.
- **Automation**: Instantly alerts humans for critical bugs/issues, significantly reducing mean time to response (MTTR) for high-priority items.

---

## Node Setup & Configuration

### 1. Webhook (Trigger)
- **Method**: `POST`
- **Path**: `ask-triage` (Production URL: `http://localhost:5678/webhook/ask-triage`)
- **Respond**: `Using 'Respond to Webhook' Node`

### 2. AI Model Node (Basic LLM Chain / Gemini / OpenAI)
- **Temperature**: `0.1`
- **Prompt**:
  ```text
  You are an AI task triage assistant.
  Return ONLY a valid minified JSON object with the following keys:
  priority (one of: urgent, high, normal, low)
  category (one of: bug, feature, content, ops, meeting, admin, other)
  owner_suggestion (string, who should handle this)
  deadline_iso (string ISO date or null)
  next_action (string, immediate next step, max 20 words)
  summary (string, brief 1-sentence summary of the task)
  confidence (0-1 number)

  Rules:
  - Never output markdown
  - Never output explanation
  - Analyze the input context to determine priority.

  User Input:
  Request text: {{$json.body.text}}
  Source: {{$json.body.source}}
  Requester: {{$json.body.requester}}
  ```

### 3. Code Node (Validation)
- **Mode**: `Run Once for All Items`
- **Language**: JavaScript
- **Code**:
  ```javascript
  return $input.all().map(item => {
    function findJsonString(obj) {
      if (typeof obj === 'string' && obj.includes('{')) return obj;
      if (obj && typeof obj === 'object') {
        for (let key in obj) {
          let found = findJsonString(obj[key]);
          if (found) return found;
        }
      }
      return null;
    }

    let rawAiOutput = findJsonString(item.json) || "";
    rawAiOutput = rawAiOutput.replace(/```json/g, "").replace(/```/g, "").trim();

    let d;
    try {
      d = JSON.parse(rawAiOutput);
    } catch (e) {
      d = { priority: "normal", summary: "Failed to parse AI output" };
    }

    const pick = (v, allowed, fallback) => (allowed && allowed.includes(v)) ? v : fallback;

    return {
      json: {
        priority: pick(d.priority, ["urgent","high","normal","low"], "normal"),
        category: pick(d.category, ["bug","feature","content","ops","meeting","admin","other"], "other"),
        owner_suggestion: (d.owner_suggestion || "unassigned").toString(),
        deadline_iso: d.deadline_iso || null,
        next_action: (d.next_action || "Review manually").toString(),
        summary: (d.summary || d.text || "No summary found").toString(),
        confidence: typeof d.confidence === "number" ? d.confidence : 0.5
      }
    };
  });
  ```

### 4. IF Node (Routing Logic)
**Condition (String)**:
- `{{ $json.priority }}` **is equal to** `urgent`
- *(Optional: Add OR condition for `high` priority)*

### 5. Action Nodes
- **True Branch (Urgent)**: Send Telegram Message.
  - Template:
    ```text
     [{{$json.priority}}] {{$json.category}} - {{$json.summary}} | action: {{$json.next_action}}
    ```
- **False Branch (Normal/Low)**: Google Sheets / Notion row addition (Auto-Resolve path).

### 6. Respond to Webhook (End)
- Connect BOTH the True and False paths to this node.
- **Respond With**: `All Incoming Item Data`

---

## Testing (Production Mode)

Ensure the workflow **Active** toggle is ON. Run these `curl` commands in your terminal to demonstrate the bot's capabilities:

**1. The Urgent Bug (Triggers Telegram Alert):**
```bash
curl -X POST "http://localhost:5678/webhook/ask-triage" -H "Content-Type: application/json" -d '{"text": "Production login is down, fix immediately.", "source": "intern-demo", "requester": "Executive"}'
```

**2. The Normal Request (Silent Triage / Backlog):**
```bash
curl -X POST "http://localhost:5678/webhook/ask-triage" -H "Content-Type: application/json" -d '{"text": "Can someone draft a social post for next Tuesday?", "source": "intern-demo", "requester": "Marketing"}'
```