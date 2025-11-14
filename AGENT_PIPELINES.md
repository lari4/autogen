# AutoGen Agent Pipelines Documentation

This document describes the various agent workflow schemes and pipelines in the AutoGen framework, showing how prompts are chained together and what data flows between each step.

## Table of Contents

1. [MagenticOne Orchestration Pipeline](#magneticone-orchestration-pipeline)
2. [Society of Mind Pipeline](#society-of-mind-pipeline)
3. [Selector Group Chat Pipeline](#selector-group-chat-pipeline)
4. [Task-Centric Memory Learning Pipeline](#task-centric-memory-learning-pipeline)
5. [Web Surfer Interaction Pipeline](#web-surfer-interaction-pipeline)
6. [.NET Development Team Pipeline](#net-development-team-pipeline)

---

## MagenticOne Orchestration Pipeline

**Purpose:** Multi-agent orchestration system that coordinates a team of specialized agents to solve complex tasks through structured planning, execution, and adaptive replanning.

**Location:** `python/packages/autogen-agentchat/src/autogen_agentchat/teams/_group_chat/_magentic_one/`

### Pipeline Flow

```
                        ┌─────────────────────────────────┐
                        │    User Task Received           │
                        └──────────────┬──────────────────┘
                                       │
                                       ▼
                        ┌──────────────────────────────────┐
                        │  PHASE 1: FACT GATHERING         │
                        │  Prompt: TASK_LEDGER_FACTS       │
                        │  Input: {task}                   │
                        │  Output: Fact sheet with:        │
                        │    - Given facts                 │
                        │    - Facts to look up            │
                        │    - Facts to derive             │
                        │    - Educated guesses            │
                        └──────────────┬───────────────────┘
                                       │
                                       ▼
                        ┌──────────────────────────────────┐
                        │  PHASE 2: PLANNING               │
                        │  Prompt: TASK_LEDGER_PLAN        │
                        │  Input: {team}, facts from P1    │
                        │  Output: Bullet-point plan       │
                        └──────────────┬───────────────────┘
                                       │
                                       ▼
                        ┌──────────────────────────────────┐
                        │  PHASE 3: EXECUTION LOOP         │
                        │  Prompt: TASK_LEDGER_FULL        │
                        │  Context: {task, team, facts,    │
                        │           plan}                  │
                        └──────────────┬───────────────────┘
                                       │
                                       ▼
                    ┌──────────────────────────────────────┐
                    │  DECISION POINT:                     │
                    │  Prompt: PROGRESS_LEDGER             │
                    │  Evaluates:                          │
                    │    - Is request satisfied?           │
                    │    - In a loop?                      │
                    │    - Making progress?                │
                    │    - Who speaks next?                │
                    │  Output: JSON decision               │
                    └──┬─────────┬─────────┬───────────────┘
                       │         │         │
           ┌───────────┘         │         └────────────┐
           │                     │                      │
           ▼                     ▼                      ▼
    ┌─────────────┐    ┌─────────────────┐    ┌──────────────┐
    │  COMPLETE   │    │  NOT MAKING     │    │  IN PROGRESS │
    │             │    │  PROGRESS       │    │              │
    └──────┬──────┘    └────────┬────────┘    └──────┬───────┘
           │                    │                     │
           │                    ▼                     │
           │         ┌──────────────────────┐         │
           │         │  REPLAN PHASE        │         │
           │         │  Sub-pipeline:       │         │
           │         │                      │         │
           │         │  1. Facts Update     │         │
           │         │     Prompt:          │         │
           │         │     FACTS_UPDATE     │         │
           │         │     Input: {task,    │         │
           │         │            facts}    │         │
           │         │     Output: Updated  │         │
           │         │            fact sheet│         │
           │         │                      │         │
           │         │  2. Plan Update      │         │
           │         │     Prompt:          │         │
           │         │     PLAN_UPDATE      │         │
           │         │     Input: {team}    │         │
           │         │     Output: New plan │         │
           │         └──────────┬───────────┘
           │                    │
           │                    └────────┐
           │                             │
           │         ┌───────────────────┘
           │         │
           │         ▼
           │    ┌──────────────────────────┐
           │    │  Select Next Speaker     │
           │    │  Execute Agent Action    │
           │    │  Update Message Thread   │
           │    └──────────┬───────────────┘
           │               │
           │               └────────┐
           │                        │
           └────────────────────────┤
                                    │
                                    ▼
                         ┌──────────────────┐
                         │  Loop back to    │
                         │  PROGRESS_LEDGER │
                         └──────────────────┘
                                    │
                                    │ (when complete)
                                    ▼
                         ┌──────────────────────┐
                         │  FINAL ANSWER        │
                         │  Prompt: FINAL_      │
                         │          ANSWER      │
                         │  Input: {task},      │
                         │         conversation │
                         │  Output: User-facing │
                         │          response    │
                         └──────────────────────┘
```

### Data Flow

1. **Initial Input:** User task description
2. **Facts Phase:** Task → FACTS_PROMPT → Structured fact sheet (4 categories)
3. **Planning Phase:** (Team + Facts) → PLAN_PROMPT → Execution plan
4. **Full Context:** (Task + Team + Facts + Plan) → FULL_PROMPT → Context for execution
5. **Progress Loop:**
   - Input: (Task + Team + Conversation history) → PROGRESS_LEDGER → JSON decision
   - Branches:
     - **Complete:** → FINAL_ANSWER → Response to user
     - **Stuck:** → (FACTS_UPDATE + PLAN_UPDATE) → Updated context → Continue
     - **In Progress:** → Select speaker → Agent acts → Update history → Loop back
6. **Output:** Final synthesized answer

### Key Characteristics

- **Adaptive:** Replans when stuck in loops or not making progress
- **Structured:** JSON outputs for machine-parseable decisions
- **Context-aware:** Maintains full task ledger throughout execution
- **Multi-stage:** Separates analysis, planning, and execution phases
- **Self-monitoring:** Continuously evaluates own progress

**Implementation:** `_magentic_one_orchestrator.py`

---

## Society of Mind Pipeline

**Purpose:** Hierarchical agent pattern where an outer agent uses an inner team of agents to generate responses, then synthesizes their work into a clean output. Useful for complex multi-step reasoning that should appear as a single response.

**Location:** `python/packages/autogen-agentchat/src/autogen_agentchat/agents/_society_of_mind_agent.py`

### Pipeline Flow

```
                    ┌────────────────────────────┐
                    │  User Message Received     │
                    │  (to Society of Mind Agent)│
                    └──────────────┬─────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │  Pass Message to Inner Team  │
                    │  (Inner team operates        │
                    │   independently with its     │
                    │   own orchestration)         │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
              ┌────────────────────────────────────────┐
              │  INNER TEAM EXECUTION                  │
              │  (Could be RoundRobin, Selector,       │
              │   MagenticOne, etc.)                   │
              │                                        │
              │  Multiple agents collaborate until     │
              │  termination condition is met          │
              └──────────────┬─────────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────────────┐
              │  Inner Team Completes                │
              │  Collected: Full conversation        │
              │            transcript from inner     │
              │            team                      │
              └──────────────┬───────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────────────┐
              │  SYNTHESIS PHASE                     │
              │  Prompt Construction:                │
              │                                      │
              │  1. System: INSTRUCTION              │
              │     "Earlier you were asked to       │
              │      fulfill a request. You and      │
              │      your team worked diligently..." │
              │                                      │
              │  2. Conversation transcript from     │
              │     inner team                       │
              │                                      │
              │  3. System: RESPONSE_PROMPT          │
              │     "Output a standalone response    │
              │      without mentioning the          │
              │      intermediate discussion"        │
              └──────────────┬───────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────────────┐
              │  LLM Call with Model Client          │
              │  Input: [Instruction + Transcript +  │
              │          Response Prompt]            │
              │  Output: Clean, synthesized response │
              └──────────────┬───────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────────────┐
              │  Reset Inner Team                    │
              │  (Prepares for next invocation)      │
              └──────────────┬───────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────────────┐
              │  Return Response to Outer Context    │
              │  (User sees only final answer,       │
              │   not inner deliberation)            │
              └──────────────────────────────────────┘
```

### Data Flow

1. **Input:** User message/task
2. **Inner Team Execution:**
   - Message → Inner Team (operates until termination)
   - Generates: Complete conversation history
3. **Synthesis:**
   - Context construction:
     - INSTRUCTION prompt (sets context)
     - Full inner team transcript
     - RESPONSE_PROMPT (requests clean output)
   - LLM processes: [Context] → Clean response
4. **Cleanup:**
   - Inner team reset (via `team.reset()`)
5. **Output:** Standalone response (no mention of internal process)

### Key Characteristics

- **Encapsulation:** Hides internal deliberation from outer context
- **Reusable:** Inner team resets after each use
- **Flexible:** Works with any team type as inner team
- **Clean Output:** Synthesizes complex discussions into simple answers
- **Stateless:** Each invocation starts fresh

### Example Use Cases

- **Multi-expert consultation:** Inner team has writer + editor, outer agent presents final draft
- **Chain-of-thought hiding:** Inner team reasons step-by-step, outer agent presents conclusion
- **Parallel exploration:** Inner team explores multiple approaches, outer agent picks best

**Implementation:** `_society_of_mind_agent.py`

---

## Selector Group Chat Pipeline

**Purpose:** Dynamic speaker selection in multi-agent conversations using LLM-based routing. The AI selects the most appropriate agent to speak next based on conversation context and agent roles.

**Location:** `python/packages/autogen-agentchat/src/autogen_agentchat/teams/_group_chat/_selector_group_chat.py`

### Pipeline Flow

```
                    ┌───────────────────────────┐
                    │  Message Arrives          │
                    │  (from user or agent)     │
                    └──────────┬────────────────┘
                               │
                               ▼
                    ┌──────────────────────────────┐
                    │  Update Message Thread       │
                    │  (Add to conversation        │
                    │   history)                   │
                    └──────────┬───────────────────┘
                               │
                               ▼
                    ┌──────────────────────────────┐
                    │  Check Termination Condition │
                    └──┬───────────────────────┬───┘
                       │                       │
                   Terminate              Continue
                       │                       │
                       ▼                       ▼
              ┌────────────┐    ┌──────────────────────────┐
              │  End       │    │  SPEAKER SELECTION       │
              └────────────┘    │                          │
                                │  Candidate Function      │
                                │  (Optional):             │
                                │  Filter available        │
                                │  speakers based on       │
                                │  context                 │
                                └──────────┬───────────────┘
                                           │
                                           ▼
                                ┌──────────────────────────┐
                                │  Build Selection Prompt  │
                                │                          │
                                │  Template: SELECTOR_     │
                                │           PROMPT         │
                                │  Variables:              │
                                │    {roles} - Agent       │
                                │             descriptions │
                                │    {participants} - List │
                                │    {history} - Recent    │
                                │                messages  │
                                └──────────┬───────────────┘
                                           │
                                           ▼
                                ┌──────────────────────────┐
                                │  LLM Selection Call      │
                                │  "You are in a role play │
                                │   game. Select the next  │
                                │   role to play..."       │
                                │                          │
                                │  Output: Agent name      │
                                └──────────┬───────────────┘
                                           │
                                           ▼
                                ┌──────────────────────────┐
                                │  Custom Selector Func    │
                                │  (Optional):             │
                                │  Override or validate    │
                                │  LLM selection           │
                                └──────────┬───────────────┘
                                           │
                                           ▼
                                ┌──────────────────────────┐
                                │  Validate Selection      │
                                │  - Is agent valid?       │
                                │  - Repeated speaker?     │
                                └──┬─────────────────┬─────┘
                                   │                 │
                               Valid            Invalid
                                   │                 │
                                   │                 ▼
                                   │      ┌──────────────────┐
                                   │      │  Retry Selection │
                                   │      │  (up to max      │
                                   │      │   attempts)      │
                                   │      └────────┬─────────┘
                                   │               │
                                   └───────────────┘
                                           │
                                           ▼
                                ┌──────────────────────────┐
                                │  Publish SelectorEvent   │
                                │  (shows selection        │
                                │   reasoning)             │
                                └──────────┬───────────────┘
                                           │
                                           ▼
                                ┌──────────────────────────┐
                                │  Invoke Selected Agent   │
                                │  (Agent processes        │
                                │   message, generates     │
                                │   response)              │
                                └──────────┬───────────────┘
                                           │
                                           ▼
                                ┌──────────────────────────┐
                                │  Loop back to Update     │
                                │  Message Thread          │
                                └──────────────────────────┘
```

### Data Flow

1. **Input:** Message from user or agent
2. **Context Building:**
   - Conversation history
   - Agent roles and descriptions
   - Available participants (optionally filtered by candidate function)
3. **Selection:**
   - Template variables: {roles, participants, history} → SELECTOR_PROMPT
   - LLM call → Agent name
   - Optional custom function validation/override
   - Retry logic if invalid (up to max attempts)
4. **Execution:**
   - Selected agent invoked
   - Agent generates response
5. **Loop:** Back to message handling

### Key Characteristics

- **Context-aware:** Selection based on full conversation history
- **Role-play framing:** Natural turn-taking through game metaphor
- **Customizable:** Optional candidate and selector functions
- **Robust:** Retry logic for invalid selections
- **Transparent:** Emits selection events showing reasoning

### Comparison with RoundRobin

| Feature | SelectorGroupChat | RoundRobinGroupChat |
|---------|------------------|---------------------|
| Speaker selection | LLM-based, context-aware | Fixed circular order |
| Flexibility | High - adapts to conversation | Low - predetermined |
| Overhead | LLM call per turn | Minimal |
| Use case | Dynamic discussions | Structured protocols |

**Implementation:** `_selector_group_chat.py`

---

## Task-Centric Memory Learning Pipeline

**Purpose:** Learning system that analyzes task failures to extract reusable insights and stores them for future task execution. Uses a teacher-student framework to build organizational memory.

**Location:** `python/packages/autogen-ext/src/autogen_ext/experimental/task_centric_memory/`

### Pipeline Flow

```
                    ┌─────────────────────────────┐
                    │  Task Execution Failed      │
                    │  Data:                      │
                    │    - Task description       │
                    │    - Work history           │
                    │    - Incorrect answer       │
                    │    - Expected answer        │
                    └──────────┬──────────────────┘
                               │
                               ▼
              ┌────────────────────────────────────────┐
              │  LEARNING PHASE: learn_from_failure()  │
              │                                        │
              │  Step 1: Review Work                   │
              │  Prompt: "You are a patient teacher"   │
              │  Input: Task + Expected + Actual +     │
              │         Work history                   │
              │  Question: "Explain what students      │
              │            did right and wrong"        │
              │  Output: Detailed analysis             │
              └──────────┬─────────────────────────────┘
                         │
                         ▼
              ┌──────────────────────────────────────┐
              │  Step 2: Identify Misconception      │
              │  Prompt: "What misconception led to  │
              │          the incorrect answer?"      │
              │  Output: Root cause analysis         │
              └──────────┬───────────────────────────┘
                         │
                         ▼
              ┌──────────────────────────────────────┐
              │  Step 3: Formulate Advice            │
              │  Prompt: "Express insights as short  │
              │          general advice (1-2 sent)"  │
              │  Output: Concise insight string      │
              └──────────┬───────────────────────────┘
                         │
                         ▼
              ┌──────────────────────────────────────┐
              │  INDEXING PHASE                      │
              │  find_index_topics()                 │
              │                                      │
              │  Prompt: "Extract task-completion    │
              │          topics"                     │
              │  Input: Insight text                 │
              │  Output: List of topic keywords      │
              └──────────┬───────────────────────────┘
                         │
                         ▼
              ┌──────────────────────────────────────┐
              │  GENERALIZATION PHASE                │
              │  generalize_task()                   │
              │                                      │
              │  Sub-pipeline:                       │
              │  1. Extract important points         │
              │     "Rephrase in general terms"      │
              │  2. Identify irrelevant points       │
              │  3. Create final general list        │
              │     "Only critical, general terms"   │
              │                                      │
              │  Output: Generalized task            │
              │          description                 │
              └──────────┬───────────────────────────┘
                         │
                         ▼
              ┌──────────────────────────────────────┐
              │  STORAGE                             │
              │  Store in memory:                    │
              │    - Insight text                    │
              │    - Topics (for retrieval)          │
              │    - Generalized task (for matching) │
              │    - Original context                │
              └──────────┬───────────────────────────┘
                         │
                         ▼
                    ┌───────────────┐
                    │  Insight Saved│
                    └───────────────┘


    ═══════════════════════════════════════════════════════════
                    RETRIEVAL PIPELINE (Separate)
    ═══════════════════════════════════════════════════════════

                    ┌─────────────────────────────┐
                    │  New Task Received          │
                    └──────────┬──────────────────┘
                               │
                               ▼
              ┌────────────────────────────────────┐
              │  Generalize New Task               │
              │  generalize_task()                 │
              │  Output: General task description  │
              └──────────┬─────────────────────────┘
                         │
                         ▼
              ┌──────────────────────────────────────┐
              │  Search Memory by Topics             │
              │  (Semantic similarity / keyword      │
              │   matching)                          │
              │  Output: Candidate insights          │
              └──────────┬───────────────────────────┘
                         │
                         ▼
              ┌──────────────────────────────────────┐
              │  VALIDATION PHASE                    │
              │  For each candidate insight:         │
              │                                      │
              │  validate_insight()                  │
              │  Prompt: "Could this insight help    │
              │          solve the task? Reply 1/0"  │
              │  Input: Insight + New task           │
              │  Output: Boolean                     │
              └──────────┬───────────────────────────┘
                         │
                         ▼
              ┌──────────────────────────────────────┐
              │  Filter Valid Insights               │
              │  Keep only insights marked '1'       │
              └──────────┬───────────────────────────┘
                         │
                         ▼
              ┌──────────────────────────────────────┐
              │  Provide to Agent                    │
              │  "Here are relevant insights from    │
              │   past experiences..."               │
              └──────────────────────────────────────┘
```

### Data Flow

**Learning (After Failure):**
1. **Input:** Task details + failure data
2. **Analysis:** Multi-step chain
   - Review (what went wrong)
   - Misconception (root cause)
   - Advice (actionable insight)
3. **Indexing:** Insight → Topics list
4. **Generalization:** Task → General description
5. **Storage:** [Insight, Topics, General task] → Memory DB

**Retrieval (Before New Task):**
1. **Input:** New task description
2. **Generalization:** Task → General form
3. **Search:** General task + Topics → Candidate insights
4. **Validation:** Each insight checked for relevance
5. **Output:** Filtered relevant insights → Agent

### Key Characteristics

- **Multi-stage learning:** Three-step chain for insight extraction
- **Semantic indexing:** Topic-based retrieval
- **Validation gate:** LLM filters irrelevant memories
- **Generalization:** Tasks abstracted to improve matching
- **Teacher-student framing:** Makes analysis more thorough

### Memory Lifecycle

1. **Failure** → Analysis → Insight → Storage
2. **New Task** → Retrieval → Validation → Augmented prompt
3. **Success/Failure** → Potentially new insights

**Implementation:** `_prompter.py`, memory storage system

