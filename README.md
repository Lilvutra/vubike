# Build an AI Agent for Vubike

## Overview

### Problem Statement

Motorbike marketplaces already solve browsing and listing discovery reasonably well.

However, real users still struggle with:

- deciding which bike fits their actual needs
- comparing multiple listings efficiently
- negotiating with sellers
- coordinating inspections
- evaluating trust and paperwork risks
- managing fragmented conversations across days/weeks

Traditional marketplaces assume users:

- know exactly what they want
- can evaluate listings rationally
- can detect scams or negotiation friction themselves

In practice, the buying journey is often uncertain, fragmented, and trust-sensitive.

The goal of this system is therefore not simply:

```text
search → click listing
```

but instead:

```text
understand → guide → coordinate → de-risk → convert
```

---

## Product Philosophy

This AI system is designed as:

```text
transaction orchestration + decision support
```

rather than:

```text
a general-purpose chatbot
```

The agent focuses on:

- workflow progression
- negotiation support
- trust/risk detection
- continuity across shopping journeys
- reducing transaction friction

---

## User Behavior Assumptions

For motorbike marketplaces, conversations are usually:

- short
- task-oriented
- episodic

Typical examples:

```text
“Show me Honda Winner X under 35M”
“Is this bike still available?”
“Can I negotiate?”
“Compare these two listings”
“Where can I inspect it?”
```

Most users do not maintain long emotional conversational threads.

However, continuity still matters because users often return during a buying journey.

Example:

| Day | User Behavior |
|---|---|
| Day 1 | Browse scooters |
| Day 3 | Ask about financing |
| Day 5 | Revisit shortlisted listings |
| Day 8 | Compare seller trustworthiness |

This means the system benefits more from:

- lightweight continuity
- operational memory
- preference persistence

rather than:

```text
infinite conversational memory
```

---

# Dataset

## Input File

```text
chat_history.jsonl
```

Each message contains:

```json
{
  "conversation_id": "c1",
  "timestamp": "2026-02-05T09:00:10Z",
  "sender": "buyer",
  "text": "Chào bạn, mình tìm xe tay ga tầm 25tr ở HCM"
}
```

---

## Fields

| Field | Description |
|---|---|
| conversation_id | unique conversation |
| timestamp | message timestamp |
| sender | buyer / seller / agent |
| text | raw unstructured message |

---

## Example Conversation Patterns

### c1 — Negotiation Friction

Buyer budget:

```text
25–26M
```

Seller expectation:

```text
32M
```

Detected signals:

- price mismatch
- negotiation tension
- seller unwilling to reduce price

---

### c2 — Document Risk

Seller states:

```text
“chưa sang tên được ngay”
```

Detected signals:

- paperwork uncertainty
- ownership transfer delay
- trust/safety escalation candidate

---

### c3 — Seller Rejects Intermediary

Seller states:

```text
“không muốn qua trung gian”
```

Detected signals:

- seller resistance to platform coordination
- bridge creation friction
- reduced orchestration capability

---

# High-Level Architecture

```text
User Message
    ↓
Normalize
    ↓
Event Type Logging
    ↓
Extraction Layer
    ↓
Operational State Update
    ↓
Policy Reasoning Engine
    ↓
Tool Orchestration(Tool Adapter/Tool Schema)
    ↓
Tool Execution
    ↓
State Transition
    ↓
Logging + Runtime Evaluation
    ↓
Agent Response
    ↓
Outcome Tracking
    ↓
Offline Evaluation
    ↓
Feedback Loop
(All stages emit Event Logs)
```
![High level architecture](agent_vubike.drawio.svg)
---

# Core Design Philosophy

The system separates memory into multiple layers.

| Layer | Purpose |
|---|---|
| Raw Logs | Source of truth stored for replayability, debugging, auditing, retraining including event logging, state update events, tool call events, tool result events, agent action events|
| Structured Memory | Reusable operational facts for retrieval and reasoning including user preferences, shpping journey state, trust/risk signals, operational coordination stage, long-term behavioural preferences, compact conversation memory|
| Rolling Summary | Compressed workflow memory for efficient LLM context to capture tension and narrative|

---

# Why This Design?

The system intentionally avoids:

```text
“store everything forever in prompt context”
```

because marketplace conversations are:

- repetitive
- localized
- operationally focused

Instead, we optimize for:

- compact workflow memory
- continuity across shopping journeys
- efficient reasoning
- observability
- replayability

---

# Memory Strategy

## What Should Be Stored?

### Structured Operational Memory

Useful reusable facts:

- preferred brands
- budget range
- city/location
- shortlisted bikes
- negotiation stage
- financing preference
- trust/risk signals

---

## What Should Stay Only In Logs?

Low-value long-term data:

- small talk
- repeated confirmations
- verbose negotiation back-and-forth
- temporary conversational noise

User message metadata
Tool invocation
Tool success/failure
State changes(historical records)



Logs remain useful for:

- replayability
- debugging
- analytics
- offline evaluation

---

# Memory Retrieval Strategy

The system does not retrieve all historical logs into the LLM context.

Instead it retrieves:

- operational state
- rolling summaries
- recent relevant messages
- critical risk signals

This acts as:

```text
compact workflow memory retrieval
```

rather than:

```text
full historical replay
```

---

# Memory Compaction Strategy

The system intentionally favors lightweight shopping continuity over deep long-term conversational memory.
For marketplace workflows, most conversations are short-lived and task-oriented rather than persistent social interactions. Therefore, we only use hot memory and warm memory here 


## HOT MEMORY

Optimized for low-latency orchestration. 

Contains:

- recent active messages
- current operational state
- active listing context
- active coordination markers


Stored in Redis/in-memory cache

Examples:

- latest negotiation
- newest buyer constraint
- recent seller response

---

## WARM MEMORY

Session-level summaries and operational state, optimized for continuity across sessions that span days, weels, multiple revisits

Contains:
- structured memory
- rolling summaries
- stable user preferences
- shopping journey progress

Examples:

- shortlisted bikes
- current negotiation stage
- unresolved risks

Stored in PostgreSQL or operational databases

---

# Compression Techniques

To prevent unbounded memory growth and token explosion, the system applies several compression strategies:

## Sliding Window

Keep recent messages active while summarizing older content.

---

## Hierarchical Summarization

Instead of one giant summary:

```text
daily summaries
→ weekly summaries
→ long-term profile
```

---

## Semantic Deduplication

Repeated facts reinforce confidence rather than duplicating memory.

Example:

```json
{
  "preferred_brand": {
    "value": "Honda",
    "confidence": 0.93,
    "mention_count": 7
  }
}
```

---

## Time Decay

Some memory naturally expires over time.

Example:

```text
importance *= exp(-lambda * age)
```

---

# Infrastructure Architecture

| Component | Purpose |
|---|---|
| Redis | active session state |
| PostgreSQL | conversations, profiles, listings |
| Kafka | event streaming for asynchronous processing |
| S3 | raw logs + tool traces for cheap large_scale archival |
| Vector DB(optional) | embeddings + semantic retrieval |

---

# State Schema

## State Design Philosophy

The schema is designed as:

```text
transaction orchestration + decision support state
```

rather than:

```text
listing search memory
```

The operational state emphasizes:

- buyer ambiguity
- negotiation dynamics
- trust/risk
- workflow progression
- coordination friction
- agent reasoning

We only store fields that are:
 
- operationally useful
- expensive/unreliable to recompute
- important for future decisions

This keeps the system compact while preserving workflow continuity.

---

# Operational State Schema

```json
{
  "conversation_id": "c1",

  "participants": {
    "buyer_id": "buyer_123",
    "seller_id": "seller_789"
  },

  "lead_stage": {
    "status": "NEGOTIATION",
    "deal_closure_likelihood": "MEDIUM",
    "buyer_interest_level": "HIGH",
    "seller_engagement_level": "MEDIUM",
    "last_progress_at": "2026-02-05T09:03:00Z"
  },

  "buyer_profile": {
    "use_case": "daily commuting",

    "decision_confidence": "MEDIUM",

    "preferences": {
      "vehicle_type": "scooter",
      "budget_min": 25000000,
      "budget_max": 26000000,
      "preferred_brands": ["Honda", "Yamaha"],
      "preferred_year_min": 2020,
      "max_odo_km": 20000,
      "location": "HCM"
    },

    "latent_needs": [
      "fuel efficiency",
      "reliable resale value"
    ],

    "deal_breakers": [
      "paperwork issues",
      "high odo"
    ]
  },

  "marketplace_state": {
    "candidate_listings": [
      {
        "listing_id": "listing_ab_2021",
        "seller_id": "seller_789",

        "status": "NEGOTIATING",

        "risk_flags": [
          "PRICE_GAP"
        ],

        "interaction_flags": [
          "seller reluctant to negotiate"
        ],

        "active_search_context": {
            "query_signature": "Honda|26000000|HCM",
            "retrieved_listing_ids": [],
            "last_search_at": ""
        },

        "active_listing_detail_context": {
            "listing_id": "",
            "last_detail_fetch_at": ""
        }
      }
    ],

    "active_channel_id": "channel_123",

    "selected_listing_id": "listing_ab_2021",

    "recommended_listing_ids": [
      "listing_vision_2022",
      "listing_lead_2021"
    ],


  },

  "trust_and_safety": {
    "global_risk_flags": [],

    "escalation_status": {
      "is_escalated": false,
      "reason": null
    }
  },

  "agent_reasoning": {
    "open_questions": [
      "Can seller reduce price?"
    ],

    "missing_information": [
      "maintenance history"
    ],

    "next_best_action": {
      "action": "NEGOTIATE_PRICE",
      "reason": "Strong buyer interest but unresolved price gap"
    }
  },

  "conversation_summary": {
    "rolling_summary":
      "Buyer seeks scooter under 26m in HCM. Seller offers Air Blade 2021 at 32m. Negotiation tension remains unresolved."
  },

  "outcome_status": {
    "appointment_booked": false,
    "time":"",
    "place":"",
    "bridge_created": true,
    "escalated": false
  },

  "updated_at": "2026-02-05T09:05:00Z"
}
```
---

# Why We Do Not Rely Entirely on User Clarification

Without operational state like active_search_context and active_listing_detail_context, the agent would repeatedly ask users to reconstruct prior context:


```text
“Bạn nói xe nào vậy?”
“Chiếc Honda nào?”
“Bạn đang nói listing hôm trước à?”
```


With those operational states it will be able to immediately reason about:

- what listings were already shown,
- whether search results are stale(because marketplace data changes dynamically due to listings being sold, seller inactive, price changes, paperwork status updates)
- whether another tool call is necessary,
- whether the user is referring to a previous bike.

without replaying full logs.

This reduces unnecessary friction during the buying journey and helps preserve continuity across episodic marketplace interactions. However, this creates a critical design tradeoff:

| Too Much Memory Reuse | Too Little Memory Reuse |
|---|---|
| incorrect assumptions | repetitive questioning |
| stale workflow continuation | poor UX friction |
| wrong tool calls | lost continuity |
| unsafe automation | reduced conversion |

Therefore, the system should not blindly reuse memory but instead these operational state fields help estimate whether automatic continuation is reliable enough.


---

# Why Tool History Is Not Stored In State

Tool calls accumulate indefinitely.

Storing them directly inside operational state would make memory:

- noisy
- expensive
- operationally inefficient

Instead:

- operational marker state stays compact
- tool calls details are persisted in logs

This separation improves:

- replayability
- observability
- evaluation
- debugging

without polluting workflow memory.

Suppose buyer says:

```text
Cho mình xem thêm xe khác
```

If no operational memory exists, the agent might call:

```text
search_listings()
```
again and again with identical filters, which lead to bad UX and wasted cost. Therefore, state stores lightweight retrieval status:

```json
{
  "marketplace_state": {
    "active_search_context": {

        "query_signature": "",

        "recommended_listing_ids": [
            "listing_1",
            "listing_2"
    ]
    }
  }
}

```

Now policy reasoning can decide that same query already executed recently without storing entire tool history



---

# State Design Benefits

| Capability | Explanation |
|---|---|
| Multi-listing support | Buyers can explore multiple bikes/sellers simultaneously |
| Compact state + operational markers| Only operationally important memory is stored while avoiding expensive raw logs replay|
| Tool compatibility | Lightweight mapping into APIs |
| Future orchestration | Supports future workflows without redesign |
| Evaluation support | Structured state enables offline metrics |
| Replayability | Logs preserve execution history separately |

---

# Tool Architecture Philosophy

We intentionally avoid forcing internal state to mirror APIs exactly.

Instead, the system separates:

| Layer | Responsibility |
|---|---|
| State | business/workflow memory |
| Policy Layer | decision logic |
| Tool Adapter | state → API translation |
| Tools | execution |

This decoupling improves:

- flexibility
- maintainability
- future extensibility

---

# Tool Orchestration Flow

```text
Message
→ Extraction
→ State Update
→ Policy Reasoning
→ Tool Decision
→ Tool Execution
→ State Transition
→ Logging
```

---
# Internal Tools

The agent does not invoke tools directly from raw chat messages.

Instead, the orchestration system follows:

```text
Message
→ Extraction
→ Operational State Update
→ Policy Reasoning
→ Tool Decision
→ Tool Adapter
→ Tool Execution
→ State Transition
→ Logging
```

The tool adapter converts semantically meaningful operational state into lightweight API-aligned fields required by internal marketplace tools.

This allows the internal state to remain business-oriented while tools remain execution-oriented.

---

# Tool Mapping Philosophy

We intentionally avoid tightly coupling:

```text
internal workflow memory
```

with:

```text
tool API schemas
```

Instead:

| Layer | Responsibility |
|---|---|
| Operational State | business/workflow memory |
| Policy Layer | decision logic |
| Tool Adapter | state → API translation |
| Tools | execution |

This separation improves:

- maintainability
- flexibility
- observability
- future extensibility

---

# 1. search_listings(query)

## Purpose

Retrieve candidate listings matching buyer preferences and constraints.

---

## Trigger Conditions

Call when:

- buyer intent = vehicle discovery
- enough search constraints exist
- confidence sufficiently high

---

## Example State Conditions

```json
{
  "lead_stage.status": "DISCOVERY",
  "buyer_profile.preferences.location": "HCM",
  "buyer_profile.preferences.budget_max": 26000000
}
```

---

## State → Tool Mapping

| Operational State Field | Tool Parameter |
|---|---|
| preferred_brands | brand |
| model | model |
| preferred_year_min | year_range.min |
| budget_min/max | price_range |
| location | location |
| latent_needs | keywords |

---

## Example Mapping

### Operational State

```json
{
  "buyer_profile": {
    "preferences": {
      "preferred_brands": ["Honda", "Yamaha"],
      "model": "Mixtral",
      "budget_max": 26000000,
      "preferred_year_min": 2020,
      "location": "HCM"
    },

    "latent_needs": [
      "fuel efficient",
      "good resale value"
    ]
  }
}
```

---

### Tool Payload

```json
{
  "brand": ["Honda", "Yamaha"],
  "year_range": {
    "min": 2020
  },
  "price_range": {
    "max": 26000000
  },
  "location": "HCM",
  "keywords": [
    "fuel efficient",
    "good resale value"
  ]
}
```

---

## Expected Output

```json
{
  "listings": [
    {
      "listing_id": "listing_vision_2021",
      "seller_id": "seller_123",
      "price": 25500000
    }
  ]
}
```

---

## State Transition

```text
DISCOVERY
→ MATCHING
```

---

# 2. get_listing_detail(listing_id)

## Purpose

Retrieve detailed listing information for:

- evaluation
- negotiation support
- risk analysis

---

## Trigger Conditions

Call when:

- buyer expresses interest
- listing shortlisted
- critical information missing

---

## Example State Conditions

```json
{
  "marketplace_state": {
    "selected_listing_id": "listing_vision_2021"
  },

  "agent_reasoning": {
    "missing_information": [
      "paperwork_status"
    ]
  }
}
```

---

## State → Tool Mapping

| Operational State Field | Tool Parameter |
|---|---|
| selected_listing_id | listing_id |

---

## Example Mapping

### Operational State

```json
{
  "marketplace_state": {
    "selected_listing_id": "listing_vision_2021"
  }
}
```

---

### Tool Payload

```json
{
  "listing_id": "listing_vision_2021"
}
```

---

## Expected Output

```json
{
  "listing_id": "listing_vision_2021",
  "brand": "Honda",
  "model": "Vision",
  "year": 2021,
  "odo_km": 18000,
  "paperwork_status": "PENDING_TRANSFER"
}
```

---

## Possible Risk Detections

| Signal | Risk |
|---|---|
| missing paperwork | DOCUMENT_RISK |
| high odo | HIGH_USAGE |
| suspicious price | FRAUD_RISK |

---

## State Transition

```text
MATCHING
→ NEGOTIATION
```

or:

```text
MATCHING
→ ESCALATION
```

---

# 3. create_chat_bridge(buyer_id, seller_id, listing_id)

## Purpose

Create trusted buyer ↔ seller communication channel.

---

## Trigger Conditions

Call when:

- buyer interest high
- seller responsive
- risks acceptable
- no escalation blocking communication

---

## Example State Conditions

```json
{
  "lead_stage": {
    "buyer_interest_level": "HIGH",
    "seller_engagement_level": "HIGH"
  },

  "trust_and_safety": {
    "global_risk_flags": []
  }
}
```

---

## State → Tool Mapping

| Operational State Field | Tool Parameter |
|---|---|
| buyer_id | buyer_id |
| candidate_listings[].seller_id | seller_id |
| selected_listing_id | listing_id |

---

## Example Mapping

### Operational State

```json
{
  "participants": {
    "buyer_id": "buyer_123"
  },

  "marketplace_state": {
    "selected_listing_id": "listing_vision_2021",

    "candidate_listings": [
      {
        "listing_id": "listing_vision_2021",
        "seller_id": "seller_456"
      }
    ]
  }
}
```

---

### Tool Payload

```json
{
  "buyer_id": "buyer_123",
  "seller_id": "seller_456",
  "listing_id": "listing_vision_2021"
}
```

---

## Expected Output

```json
{
  "channel_id": "channel_789"
}
```

---

## State Transition

```text
NEGOTIATION
→ COORDINATION
```

---

## Escalation Guardrail

Will NOT create bridge if:

```json
{
  "trust_and_safety": {
    "global_risk_flags": [
      "FRAUD_RISK"
    ]
  }
}
```

unless human review occurs.

---

# 4. book_appointment(channel_id, time, place)

## Purpose

Coordinate physical inspection/test ride, the system should optimize for qualified appointments instead of maximum appointments by requiring all trigger conditions to be sufficiently true

---

## Trigger Conditions

Call when:

- buyer intent strong/ buyer explicitly expresses viewing/test-ride intent
- seller responds cooperatively
- a listing has been selected
- active communication channel already exists
- time/place agreed
- no unresolved high-severity risk


---

## Example State Conditions

```json
{
  "lead_stage": {
    "status": "NEGOTIATION",
    "buyer_interest_level": "HIGH",
    "seller_engagement_level": "HIGH"
  },
  "marketplace_state": {
    "active_channel_id": "channel_789",
    "selected_listing_id": "listing_1"
  },

  "trust_and_safety": {
    "global_risk_flags": []
  }

}
```

---

## State → Tool Mapping

| Operational State Field | Tool Parameter |
|---|---|
| active_channel_id | channel_id |
| proposed_time | time |
| proposed_place | place |

---

## Example Mapping

### Operational State

```json
{
  "marketplace_state": {
    "active_channel_id": "channel_789"
  },

  "agent_reasoning": {
    "next_best_action": {
      "action": "BOOK_APPOINTMENT"
    }
  }
}
```

---

### Tool Payload

```json
{
  "channel_id": "channel_789",
  "time": "2026-02-10T18:00:00Z",
  "place": "Nha Rong Wharf"
}
```

---

## Expected Output

```json
{
  "booking_status": "CONFIRMED"
}
```

---

## State Transition

```text
COORDINATION
→ APPOINTMENT
```

---

# 5. log_event(conversation_id, event)

## Purpose

Persist execution traces for:

- replayability
- debugging
- evaluation
- analytics
- feedback loops

---

## Logged Event Types

| Event Type | Description |
|---|---|
| USER_MESSAGE | raw incoming message |
| AGENT_ACTION | reasoning/action selection |
| TOOL_CALL | tool invocation |
| TOOL_RESULT | tool response |
| STATE_UPDATE | operational state changed |
| ESCALATION | human escalation triggered |
| HANDOFF | transferred to human operator |
| FEEDBACK | evaluation/outcome signal |

---

## Example Event

```json
{
  "conversation_id": "c1",

  "event_type": "TOOL_CALL",

  "payload": {
    "tool": "search_listings"
  },

  "timestamp": "2026-02-05T09:01:00Z"
}
```

---

# Why This Architecture Matters

The tools are not merely APIs.

They are workflow transition mechanisms that affect:

- conversation progression
- negotiation dynamics
- trust/risk handling
- next-best-action selection
- business outcomes

By separating:

```text
state
policy
tool adapter
execution
logs
```

the system becomes:

- easier to evolve
- easier to debug
- easier to evaluate
- easier to scale


---

# Unstructured Data Handling

## Message Normalization

### Raw Input

```json
{
  "conversation_id":"c1",
  "timestamp":"2026-02-05T09:00:10Z",
  "sender":"buyer",
  "text":"Honda hoặc Yamaha, đời 2020 trở lên, oách chút" 
}
```

---

### Normalized Output
Normalize into internal standard format:

```json
{
  "conversation_id":"c1",
  "role":"buyer",
  "text":"Honda hoặc Yamaha, đời 2020 trở lên, oách chút",
  "timestamp":"2026-02-05T09:00:10Z",
  "language":"vi"
}
```

---

# Raw Logs and Events Logging

## Raw Logs 

Raw logs store the exact conversational text that act as the immutable source of truth


```json
{
  "conversation_id": "c1",
  "sender": "buyer",
  "text": "Mình muốn Honda dưới 26 triệu ở HCM"
}
```


## Event Logs

Log system-generated workflow events to detect system lifecycle transitions. Logs like TOOL_CALL, STATE_UPDATE, AGENT_ACTION capture process while FEEDBACK captures outcome quality that becomes training signal:

### Log message event

```json
{
  "event_type": "USER_MESSAGE",

  "conversation_id": "c1",

  "message_id":"msg_123"
}
```

### Log STATE_UPDATE event

```json

{
  "event_type": "STATE_UPDATE",

  "conversation_id": "c1",

  "payload": {
    "updated_fields": [
      "preferred_brands",
      "budget_max",
      "location"
    ]
  }
}
```

### Log AGENT_ACTION event

```json
{
  "event_type": "AGENT_ACTION",

  "conversation_id": "c1",

  "payload": {
    "action": "CALL_SEARCH_LISTINGS",

    "reason":
      "Buyer constraints sufficient for retrieval"
  }
}
```

### Log TOOL_CALL event

Tool adapter maps state -> API payload

```json
{
  "brand": ["Honda"],
  "price_range": {
    "max": 26000000
  },
  "location": "HCM"
}
```
Then log:

```json
{
  "event_type": "TOOL_CALL",

  "tool": "search_listings",

  "conversation_id": "c1"
}
```

### Log TOOL_RESULT event

```json
{
  "event_type": "TOOL_RESULT",

  "tool": "search_listings",

  "conversation_id": "c1",

  "payload": {
    "result_count": 2
  }
}
```

### Log ESCALATION event

```json
{
  "event_type": "ESCALATION",

  "conversation_id": "c1",

  "reason": "DEPOSIT_SCAM_RISK"
}
```

### Log HANDOFF event

```json
{
  "event_type": "HANDOFF",

  "conversation_id": "c2",

  "target": "human_support"
}
```

### Log FEEDBACK

```json
{
  "event_type": "FEEDBACK",

  "payload": {
    "conversation_outcome": "DROPPED"
  }
}
```

---

# Intent Detection Strategy

## Rule-Based Extraction

Useful for:

- deterministic workflows
- high precision
- low latency
- explainability

Examples:

- “show”
- “compare”
- “book”
- “search”

---

## ML/LLM Extraction

Useful for:

- ambiguous language
- flexible interpretation
- semantic understanding

Approach:

```text
rules first
→ ML/LLM fallback
```

to balance:

- cost
- speed
- reliability

---

# Risk Detection Strategy

The system combines:

- rule-based safety checks
- lightweight anomaly scoring
- semantic extraction

Examples:

| Signal | Risk |
|---|---|
| deposit before inspection | FRAUD_RISK |
| external Telegram redirect | OFF_PLATFORM_RISK |
| VIN mismatch | IDENTITY_RISK |
| extremely low price | SCAM_RISK |

Rules are preferred for safety because they are:

- deterministic
- explainable
- auditable
- enforceable

---

# Escalation Strategy

The system balances:

```text
under-escalation
```

vs

```text
over-escalation
```

because:

- under-escalation is dangerous
- over-escalation is expensive

---

## Escalation Triggers

- legal/document issues
- fraud suspicion
- policy violations
- low-confidence reasoning
- emotionally sensitive interactions

---

## Escalation Goals

- protect users
- protect marketplace trust
- prevent unsafe automation
- maintain conversion quality

---

# Expected Agent Behaviors

| Situation | Expected Behavior |
|---|---|
| Missing constraints | Ask clarifying questions |
| Buyer intent sufficiently clear | Search listings |
| Mutual interest detected | Create buyer ↔ seller bridge |
| Buyer requests viewing | Book appointment |
| Severe risk detected | Escalate |
| Current state | Propose a reasonable next best action |
---

# Evaluation & Feedback Loop

## Success Definition                                                                                                                                                                         
The AI agent is designed as a transaction orchestration system rather than merely a conversational chatbot.
Success is defined as helping buyers and sellers reach safe, high-quality transactions
with lower friction and higher confidence through:

- discover suitable bikes
- make better decisions
- reduce uncertainty
- coordinate efficiently
- negotiate safely
- progress toward completed transactions

---

## Hypothesis

Better:

- state tracking
- risk detection
- next-action selection

will improve:

- appointment booking
- qualified matches
- transaction completion

---

# Metrics

## Task Metrics

- match success rate: 
- appointment booking rate
- escalation accuracy
- next-action correctness

---

## Quality Metrics

- slot coverage
- hallucination rate
- policy violation rate

---

## Business Metrics

- seller response retention
- time to qualified match
- drop-off after contact

---
# Error Analysis

## Case 0 - c1
### Failure
Wrong state transition. At first it looks like a discovery stage where buyer talks to AI assistant then suddenly the system behaves like buyer-seller bride already exists and seller messages appear like a direct chat

### Missing signal
Transition conditions from discovery stage to negotiation 

### Fix

To preserve orchestration consistency, transitions between communication modes should be explicit.

The system should explicitly model:

- communication mode,
- bridge creation,
- workflow stage transition,
- channel participation metadata.

Example communication modes:

| Mode | Meaning |
|---|---|
| ASSISTANT_ONLY | buyer interacts only with AI |
| MARKETPLACE_DISCOVERY | listing exploration/search phase |
| BRIDGED_NEGOTIATION | buyer ↔ seller connected |
| APPOINTMENT_COORDINATION | scheduling/logistics phase |
| ESCALATED_REVIEW | human moderation/review involved |

Before seller participation appears inside the shared conversation stream, the workflow should transition explicitly:

```text
DISCOVERY
→ create_chat_bridge()
→ NEGOTIATION
```

Example operational events:

```json
{
  "event_type": "TOOL_CALL",
  "tool": "create_chat_bridge"
}
```

```json
{
  "event_type": "STATE_UPDATE",
  "lead_stage": "NEGOTIATION"
}
```

```json
{
  "event_type": "BRIDGE_CREATED",
  "channel_id": "channel_3"
}
```

This prevents ambiguity during:

- replayability,
- state reconstruction,
- orchestration evaluation,
- analytics,
- trust/safety enforcement.

The AI agent participates in a multi-party workflow rather than owning all message flow directly, so communication topology changes must be represented operationally rather than inferred implicitly from raw chat logs.

## Case 1 - c1

### Failure 
Wrong intent undestanding -> Agent keeps negotiating despite impossible budget gap 

### Missing signal
Price flexibility estimation 

### Fix
Add negotiation_flexibility score to recommend alternative listings earlier if price flexibility estimation is low

## Case 2 - c2

### Failure 
Missing escalation -> Agent treats did not detect paperwork issue and could prematurely recommend proceeding 

### Missing signal 
Document severity classification

### Fix
Add a document severity classification layer to distinguish between:

- minor paperwork delays,
- moderate transfer friction,
- high-risk ownership/legal uncertainty.

## Case 3 - c2 

### Failure 
Missing memory - important risk information is not persisted through chat even though the information appeared in conversation. In c2, the agent texted that it would check the paperwork process and consult buyer more carefully which is a generic response without mentioning previous message/risk regarding paperwork 

A stronger response would operationally reference earlier signals:

```text
Nếu bạn ưu tiên sang tên nhanh thì mình có thể hỗ trợ tìm thêm các xe giấy tờ hoàn chỉnh hơn.
```

### Fix
convert important conversational signals into persistent operational state
so that policy layer/future reasoning to reference that state. Since we also don't want the system to overreact to every paperwork issue, we use severity classification to determine how stongly the system reacts and how the system could give alternative responsse


## Case 4 - c3

### Failure
Missing escalation - Seller immediately requests direct off-platform contact

### Risk
Loss of marketplace trust and safety visibility

### Fix
Detect bypass intent and reinforce safe transaction workflow, encourage continued in-platform coordination,
and warn users about reduced protection outside platform visibility

---

# Feedback Loop

The system should not only generate responses, but continuously improve orchestration quality through operational feedback.

Instead of treating logs as passive history, the architecture treats them as replayable workflow traces that can be analyzed to improve:

- memory design,
- extraction quality,
- tool routing,
- escalation policies,
- next-best-action selection,
- marketplace conversion.

---

## 1. Collect Outcome Signals

After important workflow transitions, the system stores lightweight outcome signals.

Examples:

| Outcome Signal | Meaning |
|---|---|
| APPOINTMENT_BOOKED | buyer and seller scheduled viewing |
| SUCCESSFUL_MATCH | qualified connection created |
| DEAL_CLOSED | transaction completed |
| USER_DROPPED | buyer stopped engaging |
| ESCALATED | human review required |
| BRIDGE_CREATED | communication channel established |

Example:

```json
{
  "conversation_id": "c1",
  "outcome": "USER_DROPPED"
}
```

These outcomes become supervision signals for orchestration evaluation.

---

## 2. Log Operational Events

The system logs all important workflow events:

- USER_MESSAGE
- AGENT_ACTION
- TOOL_CALL
- TOOL_RESULT
- STATE_UPDATE
- ESCALATION
- HANDOFF
- FEEDBACK

Example:

```json
{
  "event_type": "STATE_UPDATE",
  "changes": {
    "seller_flexibility": "LOW"
  }
}
```

The event log acts as:

- replayable workflow history,
- debugging infrastructure,
- observability layer,
- extraction audit trail.

Raw logs remain the source of truth.

---

## 3. Maintain Compact Operational Context

The runtime system maintains compact structured context that preserves operationally meaningful signals rather than full chat history.

Example:

```json
{
  "budget_max": 26000000,
  "preferred_brands": ["Honda", "Yamaha"],
  "seller_flexibility": "LOW",
  "risk_flags": ["TRANSFER_DELAY_RISK"],
  "lead_stage": "NEGOTIATION"
}
```

The goal of context is not to preserve all conversation details, but to preserve information that meaningfully influences future workflow decisions.

A field is typically included if:

- future actions depend on it,
- recomputing it repeatedly is expensive or unreliable,
- it changes orchestration behavior,
- it affects trust, negotiation, escalation, or coordination.

---

## 4. Generate Orchestration Tuples

Operational traces are converted into compact learning tuples:

```text
(context, state, action, outcome)
```

Example:

```json
{
  "context": {
    "budget_gap": "HIGH",
    "seller_flexibility": "LOW"
  },

  "state": "NEGOTIATION",

  "action": "SEARCH_ALTERNATIVE_LISTINGS",

  "outcome": "SUCCESSFUL_MATCH"
}
```

These tuples are not raw logs.

They are distilled orchestration learning signals derived from:

- structured state,
- workflow transitions,
- operational outcomes.

---

## 5. Use Outcomes To Improve The System

### A. Improve Few-Shot Examples

Successful workflows become future orchestration examples.

Example:

```text
high price gap
→ recommend alternatives early
→ successful match
```

This improves future next-best-action reasoning.

---

### B. Improve Tool Routing

The system can learn:

- when searches improve conversion,
- when negotiation usually fails,
- when clarification questions help,
- when escalation is necessary.

Example:

| Pattern | Improvement |
|---|---|
| early search with weak constraints fails | ask clarification first |
| low seller flexibility harms negotiation | recommend alternatives earlier |
| repeated paperwork concerns cause drop-offs | escalate sooner |

This improves orchestration policies over time.

---

### C. Improve Extraction & State Updates

Repeated failures reveal weak normalization logic.

Example:

Many users say:

```text
“odo thấp”
```

but structured state never updates:

```json
{
  "max_odo_km": 20000
}
```

This indicates weak semantic extraction.

Possible improvements:

- stronger regex extraction,
- improved prompts,
- new normalization rules,
- schema updates.

---

## 6. Iteratively Refine Context Fields

Operational context fields are initially hypotheses based on workflow reasoning.

Example:

```json
{
  "seller_flexibility": "LOW"
}
```

is stored because it likely affects:

- negotiation success,
- drop-off probability,
- recommendation strategy.

Over time, logged outcomes validate whether fields are truly useful.

Example:

| seller_flexibility | negotiation outcome |
|---|---|
| LOW | negotiations often fail |
| HIGH | negotiations often succeed |

This confirms that `seller_flexibility` is an important operational signal worth preserving.

Fields that do not meaningfully improve orchestration can later be removed to keep context compact and high-signal.

The context schema therefore evolves through:

```text
hypothesis
→ logging
→ evaluation
→ refinement
```

---

## Key Insight

The architecture separates:

| Layer | Purpose |
|---|---|
| Event logs | replayable source of truth |
| Structured context/state | runtime operational memory |
| Orchestration tuples | learning and evaluation signals |

This separation enables:

- observability,
- replayability,
- safer orchestration,
- iterative improvement,
- measurable workflow optimization.

The AI agent is therefore not merely a conversational model, but a continuously improvable transaction orchestration system.




---

# Limitations

The current architecture intentionally favors:

- operational simplicity
- modularity
- observability

over:

```text
maximal intelligence
```

Current limitations include:

- limited long-term personalization
- extraction/inference noise
- stale marketplace state
- simplistic multi-listing ranking
- increased infrastructure complexity

---

# Future Improvements

Potential future directions:

- negotiation ranking models
- trust scoring systems
- personalized recommendation memory
- real-time listing synchronization
- reinforcement-learning feedback loops
- seller reliability prediction
- automated follow-up orchestration

---

# Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python / FastAPI |
| LLM orchestration | LangGraph / custom orchestration |
| Structured storage | PostgreSQL |
| Session memory | Redis |
| Streaming events | Kafka |
| Raw logs | S3 |
| Frontend | React |