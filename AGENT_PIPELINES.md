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

---

## 4. Suggestion Generation Pipeline

**Purpose**: Generates 3 relevant follow-up questions based on the conversation context. Used in both agent flows and workflows to guide users with next possible questions.

**Location**: `backend/domain/workflow/internal/repo/suggest.go` (workflow), `backend/domain/agent/singleagent/internal/agentflow/suggest_prompt.go` (agent)

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     Suggestion Generation Pipeline                            │
└──────────────────────────────────────────────────────────────────────────────┘

1. Trigger Event
   │  Event: After agent/workflow response is generated
   │  Input: User message + Assistant response
   │
   ▼
2. State Preparation
   │  Create local state object:
   │    - userMessage: User's question
   │    - answer: Assistant's response
   │
   ▼
3. Lambda Pre-Processing
   │  Action: Extract and format inputs
   │  Output: {
   │    persona_input: Optional persona for suggestions (if provided)
   │  }
   │
   ▼
4. System Prompt Construction
   │  Prompt: SUGGESTION_PROMPT (workflow) or SUGGESTION_PROMPT_JINJA2 (agent)
   │  Variables:
   │    - {{ suggest_persona }}: Optional persona context
   │
   ▼
5. Message Assembly
   │  Messages: [
   │    {role: "system", content: SUGGESTION_PROMPT},
   │    {role: "user", content: userMessage},
   │    {role: "assistant", content: answer}
   │  ]
   │
   ▼
6. LLM Invocation
   │  Model: Fast model (typically)
   │  Input: System prompt + conversation
   │  Focus: Analyze only main ideas (for speed)
   │
   ▼
7. Response Parsing
   │  Raw Output: '["Question 1?", "Question 2?", "Question 3?"]'
   │  │
   │  ▼
   │  Regex Extraction: Find JSON array pattern \[(.*?)\]
   │  │
   │  ▼
   │  JSON Parse: Extract array of strings
   │  │
   │  ▼
   │  Validation: Ensure 3 questions returned
   │
   ▼
8. Output
   └─▶ ["What is the history?", "How does it work?", "What are the benefits?"]
```

### Data Flow Example

```
Input:
  User: "Tell me about the Eiffel Tower"
  Assistant: "The Eiffel Tower is an iron lattice tower located in Paris, France.
              It was constructed from 1887 to 1889 as the entrance arch for the
              1889 World's Fair. The tower is 330 meters tall and was the world's
              tallest structure until 1930. It receives millions of visitors
              annually and is one of the most recognized structures in the world..."
  Persona: "Travel guide focusing on historical landmarks"

System Prompt: (SUGGESTION_PROMPT with persona)

LLM Processing:
  → Analyze main points: location, construction, height, visitors
  → Generate relevant follow-ups

LLM Output:
  '["What is the best time to visit the Eiffel Tower?",
    "How much does it cost to go to the top?",
    "What other landmarks are nearby?"]'

Parsing:
  → Extract JSON array
  → Validate 3 questions
  → Return array

Final Output:
  [
    "What is the best time to visit the Eiffel Tower?",
    "How much does it cost to go to the top?",
    "What other landmarks are nearby?"
  ]
```

### Code Reference

- Workflow suggester: `backend/domain/workflow/internal/repo/suggest.go:35`
- Agent suggester: `backend/domain/agent/singleagent/internal/agentflow/suggest_prompt.go:24`

---

## 5. Knowledge Retrieval Pipeline

**Purpose**: Retrieves relevant information from knowledge bases using vector search and optionally Natural Language to SQL for structured data. Integrates retrieved content into agent prompts.

**Location**: `backend/domain/knowledge/service/`, `backend/domain/workflow/internal/nodes/knowledge/`

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     Knowledge Retrieval Pipeline                              │
└──────────────────────────────────────────────────────────────────────────────┘

1. Query Input
   │  Input: User query (possibly reformulated)
   │  Context: Knowledge base IDs to search
   │
   ▼
2. Query Processing
   │  IF query reformulation enabled:
   │    ├─▶ Use Query Reformulation Pipeline
   │    └─▶ Get context-aware query
   │  ELSE:
   │    └─▶ Use original query
   │
   ▼
3. Retrieval Strategy Selection
   │
   ├─▶ Path A: Vector Search (Unstructured Documents)
   │   │
   │   ▼
   │   3a. Embedding Generation
   │       │  Action: Convert query to embedding vector
   │       │  Model: Text embedding model
   │       │
   │       ▼
   │       3b. Vector Search
   │           │  Database: Milvus/VikingDB
   │           │  Action: Similarity search (top-k)
   │           │  Parameters:
   │           │    - similarity_threshold
   │           │    - top_k (default: 5-10)
   │           │
   │           ▼
   │           3c. Retrieved Documents
   │               └─▶ [{chunk_text, score, metadata}, ...]
   │
   └─▶ Path B: NL2SQL (Structured Database)
       │
       ▼
       3d. NL2SQL Pipeline (see section 6)
           │  Input: Query + table schema
           │  Output: SQL query
           │
           ▼
           3e. SQL Execution
               │  Database: MySQL
               │  Action: Execute generated SQL
               │
               ▼
               3f. Query Results
                   └─▶ [{row1}, {row2}, ...]
   │
   ▼
4. Result Aggregation
   │  Combine results from all knowledge bases
   │  Apply reranking if configured
   │
   ▼
5. Format for LLM
   │  Action: Format retrieved content for prompt injection
   │  Output Format:
   │    For documents:
   │      "Document 1: <content>
   │       Source: <metadata>
   │
   │       Document 2: <content>
   │       Source: <metadata>"
   │
   │    For database:
   │      "Query results:
   │       | Column1 | Column2 |
   │       | Value1  | Value2  |"
   │
   ▼
6. Inject into Agent Prompt
   │  Variable: {{ knowledge }}
   │  Location: Agent system prompt
   │
   ▼
7. LLM Processing
   │  System Prompt: Includes knowledge context
   │  User Query: Original question
   │  Response: Answer grounded in retrieved knowledge
   │
   ▼
8. Final Response
   └─▶ Answer with knowledge citations (optional)
```

### Data Flow Example

```
Input:
  User Query: "What are the best practices for prompt engineering?"
  Knowledge Bases: [technical_docs_kb, best_practices_kb]

Step 2: Query (already clear, no reformulation needed)

Step 3a-3c: Vector Search
  → Generate embedding for query
  → Search in technical_docs_kb:
      Results: [
        {text: "Prompt engineering involves...", score: 0.92},
        {text: "Best practices include...", score: 0.88}
      ]
  → Search in best_practices_kb:
      Results: [
        {text: "Key principles: clarity, specificity...", score: 0.90}
      ]

Step 4: Aggregation
  → Combine and rank by score
  → Top 3 results selected

Step 5: Format
  Knowledge Content:
  ```
  Document 1 (Score: 0.92):
  Prompt engineering involves crafting inputs to language models...

  Document 2 (Score: 0.90):
  Key principles: clarity, specificity, context provision...

  Document 3 (Score: 0.88):
  Best practices include providing examples, setting constraints...
  ```

Step 6: Inject into Agent
  System Prompt:
  ```
  You are an AI assistant...

  **Knowledge**
  Only when the current knowledge has content recall, answer based on:
  '''
  [Formatted knowledge from step 5]
  '''
  ```

Step 7-8: LLM generates answer using the retrieved knowledge
```

### Code Reference

- Knowledge service: `backend/domain/knowledge/service/knowledge.go`
- Retrieval implementation: `backend/domain/knowledge/service/retrieve.go`
- Workflow integration: `backend/domain/workflow/internal/nodes/knowledge/knowledge_retrieve.go`

---

## 6. Natural Language to SQL Pipeline

**Purpose**: Converts natural language queries into MySQL-standard SQL statements for querying structured databases in knowledge bases.

**Location**: `backend/infra/document/nl2sql/impl/builtin/nl2sql.go`

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                   Natural Language to SQL Pipeline                            │
└──────────────────────────────────────────────────────────────────────────────┘

1. Input Preparation
   │  Input:
   │    - messages: Natural language query description
   │    - tables: Array of table schemas
   │
   ▼
2. Schema Formatting
   │  Lambda: Convert table schemas to readable format
   │  │
   │  For each table:
   │    ├─▶ Table Name: sales_records
   │    ├─▶ Description: Sales records table
   │    ├─▶ Columns:
   │    │     | field name  | description      | field type | is required |
   │    │     | sales_id    | id of salesperson| bigint     | true        |
   │    │     | product_id  | id of product    | bigint     | false       |
   │    │     | sale_date   | sold date/time   | datetime   | false       |
   │    │     | quantity    | sold amount      | int        | false       |
   │
   ▼
3. Prompt Construction
   │  Template: nl2sql_template_jinja2.json
   │  Variables:
   │    - {{table_schema}}: Formatted schema from step 2
   │    - {{messages}}: Natural language query
   │
   ▼
4. LLM Invocation
   │  Model: Configured NL2SQL model
   │  System Prompt: NL2SQL Consultant role with examples
   │  User Prompt: Schema + query description
   │
   ▼
5. Response Parsing
   │  Raw Output:
   │  {
   │    "sql": "SELECT ...",
   │    "err_code": 0,
   │    "err_msg": "SQL query generated successfully"
   │  }
   │  │
   │  ▼
   │  Lambda: Parse JSON and extract SQL
   │  │
   │  ▼
   │  Validation:
   │    - Check err_code
   │    - If 0: Extract SQL
   │    - If error (3002/3003/3005): Return error message
   │
   ▼
6. Output
   │  Success: SQL query string
   │  Failure: Error with reason
   │
   ▼
7. SQL Execution (downstream)
   └─▶ Execute SQL in database
   └─▶ Return results to user/agent
```

### Data Flow Example

```
Input:
  messages: "Query the top salesperson by total quantity sold last month
             and their total sales amount"
  tables: [
    {
      name: "sales_records",
      comment: "Sales records table",
      columns: [
        {name: "sales_id", type: "bigint", description: "id of salesperson", nullable: false},
        {name: "product_id", type: "bigint", description: "id of product", nullable: true},
        {name: "sale_date", type: "datetime", description: "sold date and time", nullable: true},
        {name: "quantity_sold", type: "int", description: "sold amount", nullable: true}
      ]
    }
  ]

Step 2: Schema Formatting
  table_schema = "
    table name: sales_records.
    table describe: Sales records table.

    | field name    | description        | field type | is required |
    | sales_id      | id of salesperson  | bigint     | true        |
    | product_id    | id of product      | bigint     | false       |
    | sale_date     | sold date and time | datetime   | false       |
    | quantity_sold | sold amount        | int        | false       |
  "

Step 3-4: LLM invocation with NL2SQL prompt

Step 5: LLM Output
  {
    "sql": "SELECT sales_id, SUM(quantity_sold) AS total_sales
            FROM sales_records
            WHERE MONTH(sale_date) = MONTH(CURRENT_DATE - INTERVAL 1 MONTH)
              AND YEAR(sale_date) = YEAR(CURRENT_DATE - INTERVAL 1 MONTH)
            GROUP BY sales_id
            ORDER BY total_sales DESC
            LIMIT 1",
    "err_code": 0,
    "err_msg": "SQL query generated successfully"
  }

Step 6: Parse and extract SQL

Output:
  "SELECT sales_id, SUM(quantity_sold) AS total_sales FROM..."

Step 7: Execute SQL and return results to user
```

### Error Handling

```
Error Codes:
  - 0: Success - SQL generated
  - 3002: Timeout - Generation took too long
  - 3003: Schema Missing - Required table information not provided
  - 3005: Ambiguous - Query terms are unclear or ambiguous

Example Error Response:
  {
    "sql": "",
    "err_code": 3005,
    "err_msg": "The term 'last month' is ambiguous - please specify the exact date range"
  }
```

### Code Reference

- NL2SQL implementation: `backend/infra/document/nl2sql/impl/builtin/nl2sql.go`
- Prompt template: `backend/conf/prompt/nl2sql_template_jinja2.json`

---

## 7. Query Reformulation Pipeline

**Purpose**: Reformulates user queries by incorporating conversation context, making follow-up questions self-contained and clear. Used before knowledge retrieval and agent processing.

**Location**: Query reformulation is used in agent flows and knowledge retrieval paths

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     Query Reformulation Pipeline                              │
└──────────────────────────────────────────────────────────────────────────────┘

1. Conversation History
   │  Input: Array of previous messages
   │  Format: [{role: "user", content: "..."}, {role: "assistant", content: "..."}, ...]
   │
   ▼
2. Check Reformulation Need
   │  IF conversation history exists AND current query is ambiguous:
   │    └─▶ Proceed with reformulation
   │  ELSE:
   │    └─▶ Use original query (skip to step 6)
   │
   ▼
3. Message History Formatting
   │  Action: Format conversation as JSON string
   │  Output:
   │  ```json
   │  [
   │    {"role": "user", "content": "Where is the largest desert?"},
   │    {"role": "assistant", "content": "The Sahara Desert in Africa"},
   │    {"role": "user", "content": "How to get there?"}
   │  ]
   │  ```
   │
   ▼
4. Prompt Construction
   │  Template: messages_to_query_template_jinja2.json
   │  System Message: Query reformulation engineer role
   │  Variables:
   │    - {{messages}}: Formatted conversation history
   │
   ▼
5. LLM Invocation
   │  Model: Fast model (for quick reformulation)
   │  Input: System prompt + conversation history
   │  Output: Reformulated query (plain text)
   │
   ▼
6. Reformed Query
   │  Example:
   │    Original: "How to get there?"
   │    Reformed: "How to get to the Sahara Desert?"
   │
   ▼
7. Use Reformed Query
   └─▶ Pass to knowledge retrieval / agent processing
```

### Data Flow Example

```
Conversation Context:
  Turn 1:
    User: "What is the capital of France?"
    Assistant: "The capital of France is Paris."

  Turn 2:
    User: "How many people live there?"
    (Ambiguous - "there" refers to Paris from context)

Step 1-3: Format History
  messages = [
    {"role": "user", "content": "What is the capital of France?"},
    {"role": "assistant", "content": "The capital of France is Paris."},
    {"role": "user", "content": "How many people live there?"}
  ]

Step 4-5: Reformulation
  System Prompt:
  ```
  You are a professional query reformulation engineer...
  Make queries clearer, more complete, and aligned with user's intent...
  Reply in the same language as user's input.

  Example:
  Input: [conversation with "How to get there?"]
  Output: "How to get to the Sahara Desert?"
  ```

  User Prompt:
  ```
  [formatted message history]
  ```

  LLM Processing:
  → Identify "there" refers to "Paris"
  → Reformulate with explicit reference

  LLM Output:
  "How many people live in Paris?"

Step 6-7: Use Reformed Query
  → Pass "How many people live in Paris?" to knowledge retrieval
  → Retrieve population statistics for Paris
  → Generate response: "Paris has approximately 2.2 million people..."
```

### Usage Scenarios

1. **Pronoun Resolution**
   - Original: "Tell me more about it"
   - Reformed: "Tell me more about the Eiffel Tower"

2. **Implicit Context**
   - Original: "What about pricing?"
   - Reformed: "What is the pricing for Claude API?"

3. **Continuation Questions**
   - Original: "And the next one?"
   - Reformed: "What is the second largest planet in the solar system?"

### Code Reference

- Prompt template: `backend/conf/prompt/messages_to_query_template_jinja2.json`
- Used in: Knowledge retrieval, Agent flows (when chat history enabled)

---

## Summary: Pipeline Interconnections

```
                    ┌─────────────────────────────────────┐
                    │                                     │
                    │       User Input / Request          │
                    │                                     │
                    └────────────────┬────────────────────┘
                                     │
                    ┌────────────────▼────────────────────┐
                    │                                     │
                    │   Query Reformulation Pipeline      │
                    │   (if conversation context exists)  │
                    │                                     │
                    └────────────────┬────────────────────┘
                                     │
                    ┌────────────────▼────────────────────┐
                    │                                     │
                    │      Intent Detection Pipeline      │
                    │      (if workflow configured)       │
                    │                                     │
                    └──────┬─────────────────┬────────────┘
                           │                 │
           ┌───────────────▼──┐         ┌───▼──────────────────┐
           │                  │         │                      │
           │  ReAct Agent     │         │  Workflow Execution  │
           │  Flow Pipeline   │         │                      │
           │                  │         └───┬──────────────────┘
           └───────┬──────────┘             │
                   │                        │
       ┌───────────▼────────────┬───────────▼────────────┐
       │                        │                        │
       │  Knowledge Retrieval   │   Question-Answer      │
       │  Pipeline              │   Pipeline             │
       │                        │                        │
       │  ┌──────────────┐      │                        │
       │  │   NL2SQL     │      │                        │
       │  │   Pipeline   │      │                        │
       │  └──────────────┘      │                        │
       │                        │                        │
       └───────────┬────────────┴────────────┬───────────┘
                   │                         │
                   │                         │
       ┌───────────▼─────────────────────────▼───────────┐
       │                                                  │
       │          LLM Response Generation                 │
       │                                                  │
       └───────────────────────┬──────────────────────────┘
                               │
                   ┌───────────▼──────────────┐
                   │                          │
                   │  Suggestion Generation   │
                   │  Pipeline (optional)     │
                   │                          │
                   └───────────┬──────────────┘
                               │
                   ┌───────────▼──────────────┐
                   │                          │
                   │    Final Response to     │
                   │         User             │
                   │                          │
                   └──────────────────────────┘
```

### Key Integration Points

1. **Query Reformulation → All Pipelines**: Provides context-aware queries
2. **Intent Detection → Agent/Workflow Selection**: Routes to appropriate pipeline
3. **Knowledge Retrieval → Agent System Prompt**: Injects relevant information
4. **NL2SQL → Knowledge Retrieval**: Enables structured data access
5. **Question-Answer → Workflow Interrupt**: Enables multi-turn conversations
6. **Any Response → Suggestion Generation**: Provides follow-up options

### State Management

Throughout these pipelines, state is managed using:
- **Context Cache**: Stores intermediate results (e.g., knowledge retrieval results)
- **Workflow State**: Persists across interrupts (e.g., Q&A extracted fields)
- **Chat History**: Maintained for context in multi-turn conversations
- **Local State**: Temporary state within pipeline execution (e.g., suggestion generation)

