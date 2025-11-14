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

---

## 2. Intent Detection Pipeline

**Purpose**: Classifies user input into predefined intent categories to route conversations in workflow branches. Supports both standard mode (returns JSON with reasoning) and fast mode (returns only classification ID).

**Location**: `backend/domain/workflow/internal/nodes/intentdetector/`

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                       Intent Detection Pipeline                               │
└──────────────────────────────────────────────────────────────────────────────┘

1. Workflow Input
   │  Input: User message + workflow context
   │
   ▼
2. Chat History Integration (if enabled)
   │  Action: Retrieve chat history from context
   │  Processing: Insert history messages between system and user prompts
   │
   ▼
3. System Prompt Construction
   │  Prompt: SystemIntentPrompt OR FastModeSystemIntentPrompt
   │  Variables:
   │    - {{intents}}: JSON array of intent definitions
   │      Format: [
   │        {"classificationId": 1, "content": "Book a flight"},
   │        {"classificationId": 2, "content": "Check flight status"}
   │      ]
   │    - {{advance}}: Optional advanced settings (standard mode only)
   │
   ▼
4. User Message Preparation
   │  Content: Current user input
   │  Role: user
   │
   ▼
5. LLM Invocation
   │  Model: Configured intent detection model
   │  Messages: [system_prompt, history (optional), user_message]
   │  Response Format: JSON (standard) or plain text (fast mode)
   │
   ▼
6. Response Processing
   │
   │  IF Standard Mode:
   │    ├─▶ Parse JSON response
   │    ├─▶ Extract: {
   │    │     classificationId: 2,
   │    │     reason: "User wants to check flight status"
   │    │   }
   │    └─▶ Validate JSON structure
   │
   │  IF Fast Mode:
   │    ├─▶ Parse integer from response
   │    └─▶ Return: 2 (classification ID only)
   │
   ▼
7. Output to Workflow
   │  Standard Mode: {classificationId: 2, reason: "..."}
   │  Fast Mode: {classificationId: 2}
   │
   ▼
8. Workflow Branching
   └─▶ Route to corresponding workflow branch based on classificationId
```

### Data Flow Example

**Standard Mode**:
```
Input:
  User: "Is my flight to Paris on time?"
  Intents: [
    {classificationId: 1, content: "Book a flight"},
    {classificationId: 2, content: "Check flight status"},
    {classificationId: 3, content: "Cancel reservation"}
  ]

System Prompt: (SystemIntentPrompt with intents injected)

LLM Output:
  {
    "classificationId": 2,
    "reason": "User wants to check the status of their flight to Paris"
  }

Workflow Action:
  → Branch to "Check Flight Status" node
```

**Fast Mode**:
```
Input:
  User: "Is my flight to Paris on time?"
  (Same intents)

System Prompt: (FastModeSystemIntentPrompt)

LLM Output: 2

Workflow Action:
  → Branch to "Check Flight Status" node (faster, no reasoning)
```

### Code Reference

- Intent detector implementation: `backend/domain/workflow/internal/nodes/intentdetector/intent_detector.go`
- System prompts: `intent_detector.go:211` (standard), `:242` (fast mode)

---

## 3. Question-Answer Pipeline

**Purpose**: Implements interactive Q&A with parameter extraction, choice selection, and follow-up question generation. Used for conversational forms and data collection.

**Location**: `backend/domain/workflow/internal/nodes/qa/`

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                      Question-Answer Pipeline                                 │
└──────────────────────────────────────────────────────────────────────────────┘

1. Workflow Invocation
   │  Input: Question template + configuration
   │
   ▼
2. Question Rendering
   │  Action: Render question template with variables
   │  Output: Formatted question text
   │
   ▼
3. Display Question to User
   │  IF answer_by_choices:
   │    ├─▶ Show question with choice buttons
   │    └─▶ Options: ["Option A", "Option B", "Option C", "Custom"]
   │  ELSE:
   │    └─▶ Show open-ended question input
   │
   ▼
4. Workflow Interrupt
   │  Status: INTERRUPTED
   │  Waiting for user response...
   │
   ▼
5. User Response Received
   │  Resume workflow with user input
   │
   ▼
6. Response Processing Branch
   │
   ├─▶ Path A: Answer by Choices
   │   │
   │   ▼
   │   6a. Choice Matching
   │       │
   │       IF exact match with choice:
   │         └─▶ Return: {optionId: X, optionContent: "Choice text"}
   │
   │       ELSE IF custom answer:
   │         ├─▶ Semantic Matching with LLM
   │         │   Prompt: choiceIntentDetectPrompt
   │         │   Input: User response + available choices
   │         │   Output: Option ID or -1 (no match)
   │         │
   │         └─▶ Return matched option or custom response
   │
   └─▶ Path B: Direct Answer (with extraction)
       │
       ▼
       6b. Parameter Extraction Pipeline
           │
           ├─▶ First Round Extraction
           │   │  Prompt: extractSystemPrompt
           │   │  Input: Field descriptions + user response
           │   │  Output: {
           │   │    fields: {extracted_field1: value1, ...},
           │   │    question: "What is your email?" (if fields missing)
           │   │  }
           │   │
           │   ▼
           │   IF all required fields extracted:
           │     └─▶ Return extracted fields
           │
           │   ELSE:
           │     ├─▶ Generate follow-up question
           │     ├─▶ Interrupt workflow again
           │     └─▶ Wait for next user response
           │
           └─▶ Repeat until all required fields collected

7. Final Output
   │  Direct answer: {USER_RESPONSE: "...", fields: {...}}
   │  By choices: {optionId: X, optionContent: "..."}
   │
   ▼
8. Continue Workflow
   └─▶ Use extracted data in next nodes
```

### Data Flow Example

**Parameter Extraction Flow**:
```
Question: "Please provide your contact information"
Required Fields: name (string), email (string), phone (optional)

Round 1:
  User: "My name is John Smith"

  LLM Response:
  {
    "fields": {"name": "John Smith"},
    "question": "Thank you! What is your email address?"
  }

  → Workflow interrupts again

Round 2:
  User: "john.smith@example.com"

  LLM Response:
  {
    "fields": {
      "name": "John Smith",
      "email": "john.smith@example.com"
    },
    "question": "Would you like to provide a phone number? (optional)"
  }

  → User can skip or provide phone

Final Output:
  {
    USER_RESPONSE: "john.smith@example.com",
    fields: {
      name: "John Smith",
      email: "john.smith@example.com"
    }
  }
```

**Semantic Choice Matching**:
```
Question: "How would you like to proceed?"
Choices:
  1. "Book a new flight"
  2. "Modify existing booking"
  3. "Cancel reservation"

User Input: "I'd like to change my current booking"

Semantic Matching LLM:
  Prompt: choiceIntentDetectPrompt
  Analysis: User wants to modify, not book new or cancel
  Output: 2

Final Output:
  {
    optionId: 2,
    optionContent: "Modify existing booking"
  }
```

### Code Reference

- Question-Answer implementation: `backend/domain/workflow/internal/nodes/qa/question_answer.go`
- Extraction prompt: `question_answer.go:340`
- Semantic matching prompt: `question_answer.go:368`

