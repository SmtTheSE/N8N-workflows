# N8N Workflows Repository

[![n8n](https://img.shields.io/badge/n8n-automation-ff6d5a?logo=n8n)](https://n8n.io/)
[![AI Agents](https://img.shields.io/badge/AI-Agents-00d9ff)]()
[![Telegram](https://img.shields.io/badge/Telegram-integration-26A5E4?logo=telegram)]()

## Overview

This repository contains a collection of production-ready **AI-powered automation workflows** built with n8n. These workflows implement intelligent agents for task triage, escalation management, knowledge retrieval, inbox summarization, and operational KPI monitoring.

### Purpose

Automate complex business processes using AI decision-making, reducing manual intervention and improving response times for critical operations while maintaining comprehensive audit trails.

---

## Available AI Agents

| # | Agent Name | Primary Function | Status | Documentation |
|---|------------|------------------|--------|---------------|
| 1 | **Task Triage Bot** | Intelligent task categorization & routing | Production | [Details](./n8n%20workflows/workflow1-task-triage-bot.md) |
| 2 | **Agent Escalation Bot** | Automated escalation handling & notifications | Production | [Details](./n8n%20workflows/workflow2-agent-escalation-bot.md) |
| 3 | **Knowledge Agent** | Context-aware knowledge retrieval & Q&A | Production | [Details](./n8n%20workflows/workflow3-knowledge-agent.md) |
| 4 | **Inbox Summarizer Bot** | Email/message summarization & digest | Production | [Details](./n8n%20workflows/workflow4-inbox-summarizer-bot.md) |
| 5 | **Agent Ops & KPI Dashboard Bot** | Operational metrics tracking & executive reporting | Production | [Details](./n8n%20workflows/workflow5-agent-ops-kpi-dashboard-bot.md) |

---

## Quick Start

### Prerequisites

- **n8n instance** (self-hosted or cloud)
  - Minimum version: 1.0.0
  - Recommended: Latest stable release
- **Node.js** v18+ (for self-hosted n8n)
- **API Keys** for external services:
  - Google AI Studio (Gemini) or OpenAI
  - Telegram Bot Token (for notifications)
  - Google Sheets API (for logging & KPI tracking)

### Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/SmtTheSE/N8N-workflows.git
   cd N8N-workflows
   ```

2. **Import workflows into n8n**
   - Navigate to n8n editor
   - Click "Add workflow" -> "Import from File"
   - Select the desired `.json` workflow file from `n8n workflows/` directory

3. **Configure credentials**
   - Open each imported workflow
   - Update credential nodes with your API keys:
     - AI Model credentials (Gemini/OpenAI)
     - Telegram Bot credentials
     - Google Sheets credentials (if applicable)

4. **Set webhook URLs**
   - Update webhook paths in workflow settings
   - Configure external services to point to your n8n webhook URLs

5. **Activate workflows**
   - Toggle the "Active" switch in the top-right corner of each workflow
   - Test each workflow with sample data

---

## Repository Structure

```
N8N-workflows/
├── n8n workflows/
│   ├── AI Agent Escalation Bot I.json          # Workflow 2: Escalation handling
│   ├── AI Agent Ops & KPI Dashboard Bot.json   # Workflow 5: KPI monitoring
│   ├── AI Inbox Summarizer Bot.json            # Workflow 4: Email summarization
│   ├── AI knowledge Agent.json                 # Workflow 3: Knowledge retrieval
│   ├── utonomous AI Task Triage Bot.json       # Workflow 1: Task triage
│   ├── workflow1-task-triage-bot.md            # Documentation: Triage Bot
│   ├── workflow2-agent-escalation-bot.md       # Documentation: Escalation Bot
│   ├── workflow3-knowledge-agent.md            # Documentation: Knowledge Agent
│   ├── workflow4-inbox-summarizer-bot.md       # Documentation: Inbox Summarizer
│   └── workflow5-agent-ops-kpi-dashboard-bot.md # Documentation: KPI Dashboard
├── README.md                                    # This file
└── .gitignore                                   # Git ignore rules
```

---

## Architecture & Design Principles

### Core Components

1. **Webhook Triggers**
   - RESTful endpoints for external integrations
   - Support for POST requests with JSON payloads
   - Configurable paths for each agent

2. **AI Processing Layer**
   - **Models**: Gemini 2.0 Flash Lite / Gemini 1.5 Flash / OpenAI GPT
   - **Temperature**: 0.1-0.3 for deterministic outputs
   - **Output Format**: Structured JSON for programmatic consumption

3. **Validation & Error Handling**
   - JavaScript code nodes for output validation
   - Fallback mechanisms for AI failures
   - Graceful degradation on quota errors

4. **Logging & Audit Trail**
   - Structured logging to Google Sheets
   - Unified field naming across all workflows
   - Parallel logging branches to prevent blocking

5. **Notification System**
   - Telegram integration for real-time alerts
   - Conditional notifications based on priority/risk
   - Executive summaries for high-level oversight

### Design Patterns

- **Parallel Execution**: Logging runs parallel to main logic to avoid blocking
- **Structured Outputs**: All AI responses follow strict JSON schemas
- **Error Recovery**: Automatic retry logic with exponential backoff
- **Model Failover**: Switch between AI models on quota errors

---

## Key Features

### Unified Logging Standard

All workflows adhere to a common logging schema:

| Field | Type | Description |
|-------|------|-------------|
| `timestamp` | ISO 8601 | Event occurrence time |
| `workflow_id` | String | Unique workflow identifier |
| `priority` | Enum | urgent/high/normal/low |
| `category` | Enum | bug/feature/content/ops/admin/other |
| `escalated` | Boolean | Whether escalation was triggered |
| `reason` | String | Decision rationale |
| `intent` | String | Detected user intent |
| `urgency` | Number | 0-1 urgency score |
| `risk_level` | Enum | low/medium/high/critical |
| `auto_action_allowed` | Boolean | Auto-execution permission |
| `confidence` | Number | 0-1 AI confidence score |
| `summary` | String | Brief event summary |

### High Availability

- **Quota Management**: Automatic model switching on API limits
- **Redundancy**: Multiple notification paths
- **Persistence**: All events logged before notifications

### Observability

- Real-time KPI dashboards
- Daily/weekly automated reports
- Telegram alerts for anomalies
- Historical trend analysis

---

## Security Considerations

- **API Keys**: Store in n8n credentials manager (never commit to repo)
- **Webhooks**: Use authentication headers where possible
- **Data Privacy**: Avoid logging PII (Personally Identifiable Information)
- **Access Control**: Restrict n8n editor access to authorized personnel

---

## Testing Guidelines

### Pre-Testing Checklist

- [ ] All required credentials configured
- [ ] Webhook URLs accessible from external services
- [ ] Test data prepared in source systems (Google Sheets, etc.)
- [ ] Telegram bot active and accessible

### Testing Each Workflow

1. **Unit Test Individual Nodes**
   - Execute nodes step-by-step in n8n editor
   - Verify AI model responses
   - Check validation logic

2. **End-to-End Testing**
   - Trigger workflow via webhook
   - Verify complete execution path
   - Confirm logging and notifications

3. **Failure Scenario Testing**
   - Test with invalid inputs
   - Simulate API quota exhaustion
   - Verify error handling and fallbacks

---

## Troubleshooting

### Common Issues

#### 1. **Workflow not triggering**
- **Cause**: Manual Trigger node in main execution path
- **Solution**: Remove Manual Trigger or move to separate branch

#### 2. **Empty output from AI nodes**
- **Cause**: API quota exceeded (429 error)
- **Solution**: Switch to alternative model (e.g., `gemini-1.5-flash`)

#### 3. **Logging not working**
- **Cause**: Google Sheets node disabled or missing credentials
- **Solution**: Re-enable node and configure credentials

#### 4. **JSON parsing errors**
- **Cause**: AI output not strictly JSON
- **Solution**: Adjust prompt temperature or add stricter instructions

#### 5. **Webhook conflicts**
- **Cause**: Multiple workflows using same webhook path
- **Solution**: Deactivate duplicate workflows or change paths

---

## Monitoring & Maintenance

### Daily Operations

- Review Telegram notifications for escalations
- Check n8n execution logs for failures
- Monitor API quota usage

### Weekly Reviews

- Analyze KPI dashboard trends
- Review auto-resolution rates
- Identify false positives/negatives

### Monthly Maintenance

- Update AI models if newer versions available
- Refine prompts based on performance metrics
- Archive old log data

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2024-03 | Initial release with 5 core AI agents |
| 1.1 | 2024-03 | Added unified logging standard |
| 1.2 | 2024-03 | Implemented model failover for quota management |

---

## Author & Contact

**Sitt Min Thar**  
*Data Science and Software Engineering*  
Email: sittminthar005@gmail.com

---

## License

This project is proprietary and confidential. Unauthorized copying, distribution, or use is strictly prohibited.

---

## Acknowledgments

- [n8n](https://n8n.io/) - Powerful workflow automation platform
- [Google AI](https://ai.google.dev/) - Gemini AI models
- [Telegram](https://telegram.org/) - Instant messaging platform

---

## Roadmap

- [ ] Add support for multi-language processing
- [ ] Implement advanced analytics dashboard
- [ ] Create workflow templates for common use cases
- [ ] Add CI/CD pipeline for automated deployment
- [ ] Develop testing framework for workflow validation

---

**Last Updated**: March 13, 2026
