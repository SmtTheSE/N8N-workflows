# Workflow 2: AI Agent Escalation Bot – Progress Report

This advanced autonomous bot receives customer or internal requests, evaluates them using an AI LLM, and strictly determines whether to auto-resolve the issue or instantly escalate it to human review based on computed urgency and risk vectors.

## Executive Summary
- **Purpose**: Act as an autonomous frontline triage agent that enforces safety protocols.
- **AI Intelligence**: Analyzes text to determine intent, urgency, and risk level. It drafts a response and computes an "Auto-Action Allowed" boolean.
- **Autonomous Logic (The IF Node)**: The bot is hardcoded to *only* allow auto-resolution if the AI explicitly marks it as safe (`true`), the risk is NOT high, and the urgency is NOT critical.
- **Value Delivered**: 100% automated handling of low-risk administrative/inquiry tasks (e.g., invoice requests), while ensuring zero missed critical emergencies (escalating them instantly to Telegram).

---

## Architectural Breakdown (from production JSON export)

### 1. Webhook (Trigger)
- **Method**: `POST`
- **Path**: `agent-escalation` 
- **Production URL**: `http://localhost:5678/webhook/agent-escalation`

### 2. AI "Brain" (Google Gemini Chat Model)
- **Prompt Strategy**: Strict JSON enforcement.
- **Rules applied**:
  - Never output markdown/explanation.
  - If unsure, default to medium risk / false auto-action.
  - If user impact exists on urgent issues, escalate.

### 3. JavaScript Validation Layer
- Ensures the AI's output is sanitized and stripped of markdown.
- Forces data into strict enums (e.g., `intent` must be one of predefined categories).
- **Safety Override Check**: Hardcoded safety net:
  ```javascript
  //  BOT SAFETY OVERRIDE
  if (out.urgency === "urgent" || out.risk_level === "high") {
    out.auto_action_allowed = false;
    if (!out.escalation_reason) out.escalation_reason = "Safety Override: Urgent or high-risk case";
  }
  ```

### 4. Logic Gates (IF Node)
The strict gatekeeper. To bypass human review, the task must pass **ALL** conditions:
1. `auto_action_allowed` == `true`
2. `risk_level` != `high`
3. `urgency` != `urgent`

### 5. Routing Execution
- **Dangerous/Urgent Path (False)**: Triggers the Telegram Node (`#1895930444`), formatted with Intent, Urgency, Risk, Reason, and a Draft Reply.
- **Safe Path (True)**: Routes to a "Success" Set Node, preparing the AI's draft reply for the user.

### 6. Terminal Response (End)
Replies to the original request source with the sanitized `agent_decision` payload.

---

## Live Demonstration Commands

Run these `curl` commands in your terminal to demonstrate the bot's capabilities to stakeholders:

**1. The "Auto-Resolve" Path (Silent processing / Terminal JSON only):**
```bash
curl -X POST "http://localhost:5678/webhook/agent-escalation" -H "Content-Type: application/json" -d '{"text": "Can you send me a copy of my last invoice?", "source": "intern-demo", "requester": "Marketing"}'
```

**2. The "Escalate" Path (Triggers instant Telegram Alert):**
```bash
curl -X POST "http://localhost:5678/webhook/agent-escalation" -H "Content-Type: application/json" -d '{"text": "App crashes every login after update.", "source": "intern-demo", "requester": "SupportTriage"}'
```