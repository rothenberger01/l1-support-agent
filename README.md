# L1 Support Agent: Enterprise-Grade First-Line Support

Production-ready AI agent for first-line support with domain-aware reasoning, dynamic context, and policy compliance.

**Based on:** Ilya Rice's Enterprise RAG Challenge 3 (ERC3) patterns, adapted for support domain.

## 🎯 Architecture Overview

```
User Query
    ↓
[Entry Point] - Parse user + categorize domain
    ↓
[Dynamic Context Builder] - Load KB articles + policies for domain
    ↓
[Planner/Orchestrator] - SGR (Schema-Guided Reasoning) plan next steps
    ↓
[Tool Executor] - Call KB search, ticket system, escalation tools
    ↓
[Validator] - Check: answer format, policy compliance, escalation need
    ↓
User Response (structured, with confidence + escalation flag)
```

## 🚀 Key Features

### 1. **Domain-Aware Routing**
- Automatically detects support domain (billing, technical, account, etc.)
- Loads only relevant KB articles + rules for that domain
- Prevents hallucination by limiting context scope

### 2. **Dynamic Context**
- Not "dump entire KB into LLM" → Instead:
  - Lightweight semantic search for relevant articles
  - Filter by domain + user type
  - Include only essential metadata
  - Conversation history summary (not raw)
- Result: 4-8k tokens context vs potentially 100k+

### 3. **Structured Output (Not Native Function Calling)**
- All agent decisions as strict JSON schema
- Why? Reliability, auditability, easy validator integration
- Schema defines: action type, parameters, reasoning, confidence

### 4. **Validator Layer**
- Second-pass check on agent output:
  - Is answer compliant with policies?
  - Is confidence > threshold or should escalate?
  - Is format correct?
- Prevents "helpful but wrong" responses

### 5. **Escalation Logic**
- Agent learns when to escalate (not try to answer everything)
- Escalation triggers:
  - Confidence < threshold
  - Policy requires human review
  - Complex issue detected
  - Special customer flags

## 📁 Project Structure

```
l1-support-agent/
├── README.md
├── pyproject.toml                 # Python deps
├── .env.example                   # Config template
│
├── config/
│   ├── domains.yaml              # Domain definitions + rules
│   ├── policies.yaml             # Access/compliance rules
│   └── prompts.yaml              # Orchestrator + Validator prompts
│
├── src/
│   ├── __init__.py
│   ├── main.py                   # FastAPI app entry point
│   ├── models.py                 # Pydantic schemas (Request/Response/Step)
│   │
│   ├── core/
│   │   ├── __init__.py
│   │   ├── types.py              # Enums, TypedDicts
│   │   ├── state.py              # State machine + storage
│   │   └── logger.py             # Tracing (like Ilya's JSON logs)
│   │
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── kb_search.py          # KB / vector search
│   │   ├── ticket.py             # Ticket system API
│   │   ├── customer.py           # Customer data API
│   │   └── executor.py           # Tool dispatcher (validates + calls)
│   │
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── planner.py            # Orchestrator (SGR logic)
│   │   ├── validator.py          # Policy + format validator
│   │   └── context_builder.py    # Dynamic context assembly
│   │
│   └── utils/
│       ├── __init__.py
│       ├── llm.py                # LLM calls (OpenAI, Gemini, etc.)
│       ├── schemas.py            # JSON schema validators
│       └── retry.py              # Retry logic
│
├── tests/
│   ├── test_planner.py
│   ├── test_validator.py
│   ├── test_context.py
│   └── test_integration.py
│
├── examples/
│   ├── single_shot.py            # Minimal working example
│   ├── trace_viewer.py           # View execution traces
│   └── sample_tickets.json       # Test data
│
└── docs/
    ├── ARCHITECTURE.md           # Deep dive
    ├── DOMAIN_SETUP.md           # How to add new domain
    └── PROMPTS.md                # Prompt engineering guide
```

## 🔧 Quick Start

### Prerequisites
- Python 3.10+
- OpenAI API key (or Gemini/Claude)
- Optional: Postgres for state storage

### 1. Clone & Setup

```bash
git clone https://github.com/rothenberger01/l1-support-agent.git
cd l1-support-agent

python -m venv venv
source venv/bin/activate  # or `venv\Scripts\activate` on Windows

pip install -r requirements.txt
cp .env.example .env
# Edit .env with your API keys
```

### 2. Define Your Domains

Edit `config/domains.yaml`:

```yaml
billing:
  name: Billing & Payments
  keywords: [charge, invoice, refund, payment, subscription]
  kb_tags: [billing, payment, subscription]
  tools: [kb_search, ticket_create]
  escalation_triggers:
    - confidence < 0.7
    - refund_amount > 500
    - vip_customer: true
  rules:
    - never_promise_refund_without_supervisor
    - must_cite_source

technical:
  name: Technical Issues
  keywords: [error, bug, crash, doesn't work, broken]
  kb_tags: [technical, troubleshooting, faq]
  tools: [kb_search, ticket_create]
  escalation_triggers:
    - confidence < 0.8
    - mention: "critical"
    - resolve_time_sla: 1h
```

### 3. Load KB Articles

```bash
python -c "from src.tools.kb_search import load_kb; load_kb('data/kb.json')"
```

KB format (JSON):
```json
{
  "articles": [
    {
      "id": "kb-001",
      "title": "How to reset password",
      "domain": "account",
      "content": "To reset your password...",
      "tags": ["password", "account", "security"],
      "qa_pairs": [
        {"q": "I forgot my password", "a": "..."}
      ]
    }
  ]
}
```

### 4. Run Agent

```bash
uvicorn src.main:app --reload
```

Test:
```bash
curl -X POST http://localhost:8000/support \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "cust-123",
    "query": "Can I get a refund for my subscription?",
    "conversation_id": "conv-456"
  }'
```

Response:
```json
{
  "conversation_id": "conv-456",
  "step": 5,
  "final_answer": {
    "message": "Yes, you can request a refund within 30 days...",
    "domain": "billing",
    "confidence": 0.92,
    "sources": ["kb-042"],
    "escalate": false
  },
  "escalation_reason": null,
  "trace_id": "trace-789"
}
```

## 🧠 How the Agent Works (Step-by-Step)

### Step 1: Entry Point
Parse user query, detect domain, load customer context.

### Step 2: Dynamic Context Builder
- Search KB for top-3 relevant articles
- Load domain rules + policies
- Get conversation summary from state
- Assemble final context (capped at 6k tokens)

### Step 3: Planner (Orchestrator)
LLM gets:
```
You are a first-line support agent for {domain}.

[System Prompt]
- Available tools: kb_search, ticket_create, escalate
- Output ONLY valid JSON matching this schema: {...}
- Do NOT hallucinate answers
- If unsure, set confidence < 0.7 and recommend escalation

[Context]
KB Articles: {formatted_articles}
Domain Rules: {rules}
Conversation: {history}

[User Query]
{user_query}

Return JSON:
{
  "reasoning": "...",
  "action_type": "answer" | "tool_call" | "escalate",
  "answer": "...",
  "confidence": 0.0-1.0,
  "tool_calls": [...],
  "escalation_reason": "..." 
}
```

### Step 4: Tool Executor
If `action_type = "tool_call"`:
- Validate tool name + arguments
- Call tool (kb_search, etc.)
- Return result to Planner
- Loop until `action_type = "answer"`

### Step 5: Validator
Check final answer:
```python
{
  "valid": true,
  "feedback": null,
  "escalate_if_confidence_low": true
}
```

If `valid = false` or confidence needs boost → loop back to Planner.

## 🎯 Domain Configuration Deep Dive

See `docs/DOMAIN_SETUP.md` for:
- Adding new domain with custom rules
- Creating domain-specific validators
- Setting escalation thresholds
- KB tagging strategy

## 📊 Tracing & Debugging

Every request generates a JSON trace:

```json
{
  "trace_id": "trace-123",
  "conversation_id": "conv-456",
  "timestamp": "2026-01-29T21:15:00Z",
  "user_id": "cust-789",
  "domain_detected": "billing",
  "steps": [
    {
      "step_id": 1,
      "action": "context_builder",
      "duration_ms": 234,
      "context": {...}
    },
    {
      "step_id": 2,
      "action": "planner",
      "duration_ms": 1234,
      "llm_model": "gpt-4",
      "input_tokens": 1240,
      "output_tokens": 340,
      "output": {...}
    },
    {
      "step_id": 3,
      "action": "validator",
      "duration_ms": 456,
      "decision": "approve",
      "feedback": null
    }
  ],
  "final_answer": {...},
  "total_duration_ms": 1924
}
```

View traces:
```bash
python examples/trace_viewer.py --trace-id trace-123
```

## 🚀 Production Deployment

See `docs/DEPLOYMENT.md` for:
- Docker setup
- Kubernetes manifest
- Monitoring + alerting
- Cost optimization
- Rate limiting

## 📚 Learning Resources

1. **Video Reference:** [Ilya Rice - Enterprise RAG Challenge 3](https://www.youtube.com/watch?v=3JYHMMw5WSU)
2. **Ilya's Code:** [github.com/IlyaRice/Enterprise-RAG-Challenge-3-AI-Agents](https://github.com/IlyaRice/Enterprise-RAG-Challenge-3-AI-Agents)
3. **Key Concepts:**
   - SGR (Schema-Guided Reasoning) → forcing LLM into strict JSON
   - Dynamic Context → don't dump everything
   - Validator Pattern → second-pass policy check
   - State Machine → track conversation flow

## 📝 License

MIT

## 🤝 Contributing

Pull requests welcome! Focus areas:
- New domain configurations
- KB search optimization
- Validator improvements
- LLM provider integrations
