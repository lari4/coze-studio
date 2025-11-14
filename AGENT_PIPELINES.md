# Agent Pipelines Documentation

This document provides a comprehensive overview of all agent workflows and pipelines in the Coze Studio application. It shows how different prompts are chained together, what data flows between them, and the overall architecture of agent execution.

## Table of Contents

1. [Pipeline Overview](#pipeline-overview)
2. [ReAct Agent Flow Pipeline](#1-react-agent-flow-pipeline)
3. [Intent Detection Pipeline](#2-intent-detection-pipeline)
4. [Question-Answer Pipeline](#3-question-answer-pipeline)
5. [Suggestion Generation Pipeline](#4-suggestion-generation-pipeline)
6. [Knowledge Retrieval Pipeline](#5-knowledge-retrieval-pipeline)
7. [Natural Language to SQL Pipeline](#6-natural-language-to-sql-pipeline)
8. [Query Reformulation Pipeline](#7-query-reformulation-pipeline)

---

## Pipeline Overview

Coze Studio uses a sophisticated pipeline architecture based on the Eino framework (Cloudwego/eino) for composing chains of operations. Each pipeline is built using:

- **Runnables**: Composable units that can be invoked with inputs and produce outputs
- **Chains**: Sequential compositions of operations (lambdas, prompts, models)
- **Prompt Templates**: Jinja2-based templates for formatting messages
- **Chat Models**: LLM invocations with streaming support
- **State Management**: Context preservation across pipeline stages

### Common Pipeline Patterns

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌────────────┐
│   Input     │────▶│   Lambda     │────▶│   Prompt    │────▶│  LLM Model │
│ Processing  │     │ Transformation│     │  Template   │     │   Call     │
└─────────────┘     └──────────────┘     └─────────────┘     └────────────┘
                                                                      │
                                                                      ▼
                                                              ┌────────────┐
                                                              │   Output   │
                                                              │ Processing │
                                                              └────────────┘
```

---

## 1. ReAct Agent Flow Pipeline

**Purpose**: The main execution pipeline for single agents using the ReAct (Reasoning + Acting) pattern. This pipeline handles agent persona, knowledge retrieval, tool calling, and response generation.

**Location**: `backend/domain/agent/singleagent/internal/agentflow/`

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         ReAct Agent Flow Pipeline                             │
└──────────────────────────────────────────────────────────────────────────────┘

1. User Input
   │
   ▼
2. Query Reformulation (if chat history exists)
   │  Prompt: Query Reformulation Prompt
   │  Input: conversation history + current query
   │  Output: reformulated query
   │
   ▼
3. Knowledge Retrieval (if knowledge base configured)
   │  Action: Search knowledge base with query
   │  Output: relevant documents/chunks
   │
   ▼
4. Pre-Tool Retrieval (if pre-retrieval tools configured)
   │  Action: Execute configured tools before main LLM call
   │  Output: tool results
   │
   ▼
5. System Prompt Construction
   │  Prompt: REACT_SYSTEM_PROMPT_JINJA2
   │  Variables:
   │    - {{ agent_name }}: Agent name from configuration
   │    - {{ time }}: Current timestamp
   │    - {{ persona }}: Custom agent persona/instructions
   │    - {{ memory_variables }}: Memory/context variables
   │    - {{ knowledge }}: Retrieved knowledge base content
   │    - {{ tools_pre_retriever }}: Pre-tool call results
   │
   ▼
6. LLM Invocation with Tools
   │  Model: Configured chat model
   │  System Message: Constructed system prompt
   │  User Message: User input (possibly reformulated)
   │  Tools: Available function calls (plugins, workflows)
   │  Output: Response with potential tool calls
   │
   ▼
7. Tool Execution Loop
   │  IF tool calls requested:
   │    ├─▶ Execute tool (plugin/workflow)
   │    ├─▶ Collect tool results
   │    └─▶ Return to step 6 with tool results
   │  ELSE:
   │    └─▶ Continue to step 8
   │
   ▼
8. Final Response
   │  Output: Agent response to user
   │
   ▼
9. Suggestion Generation (optional)
   │  Prompt: SUGGESTION_PROMPT_JINJA2
   │  Input:
   │    - {{_input_}}: Original user query
   │    - {{_answer_}}: Agent's response
   │    - {{ suggest_persona }}: Optional persona for suggestions
   │  Output: Array of 3 follow-up questions
   │
   ▼
10. Final Output
    └─▶ Response + Suggestions delivered to user
```

### Data Flow Details

**Step 2: Query Reformulation**
- **Input**: `[{role: "user", content: "previous question"}, {role: "assistant", content: "previous answer"}, {role: "user", content: "current question"}]`
- **Processing**: Uses query reformulation prompt to make the current question self-contained
- **Output**: Reformed query (e.g., "How to get there?" → "How to get to the Sahara Desert?")

**Step 3: Knowledge Retrieval**
- **Input**: Reformulated query
- **Processing**: Vector search or hybrid search in knowledge base
- **Output**: Top-k relevant documents with similarity scores

**Step 5: System Prompt Variables**
```javascript
{
  agent_name: "Travel Assistant",
  time: "2025-11-14 10:30:00",
  persona: "You are a knowledgeable travel guide...",
  memory_variables: "User preference: beach destinations\nBudget: $5000",
  knowledge: "The Sahara Desert is located in North Africa...",
  tools_pre_retriever: "[Tool: weather_api] Current weather in Sahara: 35°C"
}
```

**Step 7: Tool Execution Loop**
```
LLM Response: "Let me search for flights to Morocco..."
  │
  ▼
Tool Call: {
  name: "search_flights",
  arguments: {
    destination: "Morocco",
    dates: "next month"
  }
}
  │
  ▼
Tool Result: [
  {flight: "Air France AF123", price: "$450"},
  {flight: "Royal Air Maroc AT456", price: "$380"}
]
  │
  ▼
Return to LLM with tool results
  │
  ▼
LLM Final Response: "I found two flights for you..."
```

### Code Reference

- Agent flow builder: `backend/domain/agent/singleagent/internal/agentflow/agent_flow_builder.go`
- System prompt: `backend/domain/agent/singleagent/internal/agentflow/system_prompt.go:27`
- Suggestion prompt: `backend/domain/agent/singleagent/internal/agentflow/suggest_prompt.go:24`

