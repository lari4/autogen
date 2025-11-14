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

