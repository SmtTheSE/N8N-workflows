# Workflow 5: AI Agent Ops & KPI Dashboard Bot – Progress Report

This workflow aggregates logs and outputs from the other AI workflows (1–4) and produces a daily or weekly operational report:

- Agent throughput (how many tasks handled by each bot)
- Escalations vs auto-resolutions
- Simple SLA-style metrics
- The most important issues and risks surfaced across all agents

It turns the raw activity of the AI agents into a single, executive-friendly KPI snapshot.

## Executive Summary
- **Purpose**: Provide a single view of how all AI agents are performing, what they are doing, and what is slipping.
- **AI Intelligence**: Reads structured logs from Workflows 1–4, computes KPIs, clusters issues, and generates a concise narrative summary plus a risk list.
- **Automation**: Runs on a schedule (e.g., once per day or once per week), writes a KPI row to a sheet/DB, and pushes a short KPI digest to Telegram.

---

## Node Setup & Configuration

### 1. Trigger – Scheduled KPI Run

- **Node**: `Cron`
- **Example schedule**:
  - Daily at 18:00, or
  - Weekly on Friday at 18:00 for a “weekly ops review”.

This is a pure backend workflow (no external webhook needed).

### 2. Data Source Nodes – Pull Logs from Each Workflow

Assuming Google Sheets logs for each workflow, add one node per source (DB/Notion variants are similar):

1. **Workflow 1 Log – Task Triage**
   - Node: `Google Sheets` → `Read`
   - Sheet: `AI_Task_Triage_Log`
   - Filter: rows where `timestamp` is in the last 24h (or last 7 days).

2. **Workflow 2 Log – Agent Escalation**
   - Node: `Google Sheets` → `Read`
   - Sheet: `AI_Agent_Escalation_Log`
   - Filter: last 24h / 7 days.

3. **Workflow 3 Log – Knowledge Agent**
   - Node: `Google Sheets` → `Read`
   - Sheet: `AI_Knowledge_Agent_Log`
   - Filter: last 24h / 7 days.

4. **Workflow 4 Log – Inbox Summarizer**
   - Node: `Google Sheets` → `Read`
   - Sheet: `AI_Inbox_Summary_Log`
   - Filter: last 24h / 7 days.

### 3. Code Node – Consolidate All Logs

- **Mode**: `Run Once for All Items`
- **Language**: JavaScript
- **Goal**: Merge rows from all four sources into a single structured payload for the AI.

Example structure:

```javascript
const triage = $items('Task Triage Log');       // Workflow 1
const escalation = $items('Escalation Log');    // Workflow 2
const knowledge = $items('Knowledge Log');      // Workflow 3
const inbox = $items('Inbox Log');              // Workflow 4

function extractJson(items) {
  return items.map(i => i.json);
}

return [
  {
    json: {
      generated_at: new Date().toISOString(),
      window: 'last_24h', // or 'last_7d'
      workflow1_triage: extractJson(triage),
      workflow2_escalation: extractJson(escalation),
      workflow3_knowledge: extractJson(knowledge),
      workflow4_inbox: extractJson(inbox),
    }
  }
];
```

### 4. AI Model Node – KPI Summary & Risk Analysis

Use Gemini/OpenAI with strict JSON output.

- **Temperature**: `0.2`
- **Prompt**:
  ```text
  You are an AI ops analyst.

  You will receive structured log data from 4 AI workflows:
  - workflow1_triage: task triage events
  - workflow2_escalation: agent escalation decisions
  - workflow3_knowledge: knowledge summaries
  - workflow4_inbox: inbox summaries

  Return ONLY a valid minified JSON object with keys:
  date (string, ISO date for this report)
  window_label (string, e.g. "last_24h" or "last_7d")
  headline (string, max 25 words)
  kpi_summary (object) with keys:
    total_tasks (number)
    triage_tasks (number)
    escalations (number)
    auto_resolved (number)
    high_importance_knowledge_items (number)
    inbox_summaries (number)
  narrative (string, max 250 words explaining what happened)
  top_incidents (array of 3-10 short strings summarizing critical incidents or themes)
  risks (array of strings, can be empty)
  improvement_suggestions (array of 3-7 short action suggestions for the team)

  Rules:
  - Never output markdown.
  - Never output explanations outside the JSON.
  - If some data is missing, estimate conservatively and mention this in the narrative.

  Here is the input JSON:
  {{$json}}
  ```

### 5. Code Node – Validate & Normalize AI Output

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

  let raw = findJsonString(item.json) || '';
  raw = raw.replace(/```json/g, '').replace(/```/g, '').trim();

  let d;
  try {
    d = JSON.parse(raw);
  } catch (e) {
    d = {
      date: new Date().toISOString().slice(0,10),
      window_label: 'unknown',
      headline: 'Failed to parse KPI summary.',
      kpi_summary: {
        total_tasks: 0,
        triage_tasks: 0,
        escalations: 0,
        auto_resolved: 0,
        high_importance_knowledge_items: 0,
        inbox_summaries: 0,
      },
      narrative: '',
      top_incidents: [],
      risks: [],
      improvement_suggestions: []
    };
  }

  const kpi = d.kpi_summary || {};
  const num = (v) => (typeof v === 'number' ? v : 0);

  return {
    json: {
      date: String(d.date || new Date().toISOString().slice(0,10)),
      window_label: String(d.window_label || 'last_24h'),
      headline: String(d.headline || ''),
      kpi_summary: {
        total_tasks: num(kpi.total_tasks),
        triage_tasks: num(kpi.triage_tasks),
        escalations: num(kpi.escalations),
        auto_resolved: num(kpi.auto_resolved),
        high_importance_knowledge_items: num(kpi.high_importance_knowledge_items),
        inbox_summaries: num(kpi.inbox_summaries),
      },
      narrative: String(d.narrative || ''),
      top_incidents: Array.isArray(d.top_incidents) ? d.top_incidents.map(String) : [],
      risks: Array.isArray(d.risks) ? d.risks.map(String) : [],
      improvement_suggestions: Array.isArray(d.improvement_suggestions) ? d.improvement_suggestions.map(String) : []
    }
  };
});
```

### 6. Logging – KPI History Sheet / DB

Append one row per run.

- **Destination**: `Google Sheets` or DB table `AI_Agent_KPI_Reports`
- **Columns**:
  - `date`
  - `window_label`
  - `headline`
  - `kpi_summary_json` (raw JSON)
  - `narrative`
  - `top_incidents_joined`
  - `risks_joined`
  - `improvement_suggestions_joined`

This becomes a long-term operations dataset for AI agents.

### 7. Telegram Notification – Ops Digest

Send a short KPI digest to Telegram:

```text
 AI Agent Ops Report ({{$json.date}} – {{$json.window_label}})

{{$json.headline}}

KPI:
- Total tasks: {{$json.kpi_summary.total_tasks}}
- Escalations: {{$json.kpi_summary.escalations}}
- Auto-resolved: {{$json.kpi_summary.auto_resolved}}

Top incidents:
- {{$json.top_incidents[0]}}
- {{$json.top_incidents[1]}}

Risks: {{ $json.risks.length > 0 ? 'see log' : 'none flagged' }}
```

(Guard indices in n8n expressions so you dont crash on short arrays.)

---

## How This Completes the Five-Workflow Set
- **Workflow 1** triages tasks.
- **Workflow 2** enforces escalation and safety.
- **Workflow 3** turns content into structured insights.
- **Workflow 4** summarizes inbox activity.
- **Workflow 5** (this one) measures and explains how the others are performing, providing a concise operational view for decision-makers.
