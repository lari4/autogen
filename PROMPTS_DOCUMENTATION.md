# AutoGen AI Prompts Documentation

This document provides comprehensive documentation of all AI prompts used in the AutoGen framework, organized by functional categories.

## Table of Contents

1. [Orchestration Prompts](#orchestration-prompts)
2. [Web Surfing Prompts](#web-surfing-prompts)
3. [Coding Assistant Prompts](#coding-assistant-prompts)
4. [Memory and Learning Prompts](#memory-and-learning-prompts)
5. [Agent Role Prompts](#agent-role-prompts)
6. [Development Team Prompts](#development-team-prompts)
7. [Tool-Specific Prompts](#tool-specific-prompts)
8. [Selection and Routing Prompts](#selection-and-routing-prompts)

---

## Orchestration Prompts

These prompts are used by the MagenticOne orchestrator to coordinate multiple agents, track progress, and manage task execution.

**Location:** `python/packages/autogen-agentchat/src/autogen_agentchat/teams/_group_chat/_magentic_one/_prompts.py`

### ORCHESTRATOR_SYSTEM_MESSAGE

**Purpose:** System message for the orchestrator agent. Intentionally left empty to allow the orchestrator to operate without predefined system-level constraints.

**Usage:** Initialized as the system message for the MagenticOne orchestrator.

```python
ORCHESTRATOR_SYSTEM_MESSAGE = ""
```

### ORCHESTRATOR_TASK_LEDGER_FACTS_PROMPT

**Purpose:** Initial analysis prompt that asks the AI to conduct a pre-survey of the task before beginning work. This helps identify what information is known, what needs to be looked up, what needs to be derived, and what can be reasonably guessed. This structured analysis helps the orchestrator make informed decisions about task execution.

**Usage:** Called at the beginning of a task to gather initial facts and understanding.

**Variables:**
- `{task}`: The user's original request

```python
ORCHESTRATOR_TASK_LEDGER_FACTS_PROMPT = """Below I will present you a request. Before we begin addressing the request, please answer the following pre-survey to the best of your ability. Keep in mind that you are Ken Jennings-level with trivia, and Mensa-level with puzzles, so there should be a deep well to draw from.

Here is the request:

{task}

Here is the pre-survey:

    1. Please list any specific facts or figures that are GIVEN in the request itself. It is possible that there are none.
    2. Please list any facts that may need to be looked up, and WHERE SPECIFICALLY they might be found. In some cases, authoritative sources are mentioned in the request itself.
    3. Please list any facts that may need to be derived (e.g., via logical deduction, simulation, or computation)
    4. Please list any facts that are recalled from memory, hunches, well-reasoned guesses, etc.

When answering this survey, keep in mind that "facts" will typically be specific names, dates, statistics, etc. Your answer should use headings:

    1. GIVEN OR VERIFIED FACTS
    2. FACTS TO LOOK UP
    3. FACTS TO DERIVE
    4. EDUCATED GUESSES

DO NOT include any other headings or sections in your response. DO NOT list next steps or plans until asked to do so.
"""
```

### ORCHESTRATOR_TASK_LEDGER_PLAN_PROMPT

**Purpose:** Creates an execution plan based on the team composition and known/unknown facts. This prompt encourages the orchestrator to think strategically about which team members to involve and in what sequence.

**Usage:** Called after facts are gathered to create the initial execution plan.

**Variables:**
- `{team}`: Description of available team members and their capabilities

```python
ORCHESTRATOR_TASK_LEDGER_PLAN_PROMPT = """Fantastic. To address this request we have assembled the following team:

{team}

Based on the team composition, and known and unknown facts, please devise a short bullet-point plan for addressing the original request. Remember, there is no requirement to involve all team members -- a team member's particular expertise may not be needed for this task."""
```

### ORCHESTRATOR_TASK_LEDGER_FULL_PROMPT

**Purpose:** Provides the complete task context to the orchestrator, combining the task, team composition, facts, and plan. This serves as the comprehensive "ledger" that guides the entire execution.

**Usage:** Used to maintain the full context of the task throughout execution.

**Variables:**
- `{task}`: The user's original request
- `{team}`: Description of available team members
- `{facts}`: The fact sheet from the initial analysis
- `{plan}`: The execution plan

```python
ORCHESTRATOR_TASK_LEDGER_FULL_PROMPT = """
We are working to address the following user request:

{task}


To answer this request we have assembled the following team:

{team}


Here is an initial fact sheet to consider:

{facts}


Here is the plan to follow as best as possible:

{plan}
"""
```

### ORCHESTRATOR_PROGRESS_LEDGER_PROMPT

**Purpose:** Evaluates the current progress of the task execution. This is the key prompt for decision-making during execution - it asks the orchestrator to assess whether the task is complete, whether the team is stuck in a loop, whether progress is being made, and who should speak next. The JSON schema ensures structured, parseable responses.

**Usage:** Called repeatedly during task execution to make routing decisions and track progress.

**Variables:**
- `{task}`: The user's original request
- `{team}`: Description of available team members
- `{names}`: List of team member names for selection

```python
ORCHESTRATOR_PROGRESS_LEDGER_PROMPT = """
Recall we are working on the following request:

{task}

And we have assembled the following team:

{team}

To make progress on the request, please answer the following questions, including necessary reasoning:

    - Is the request fully satisfied? (True if complete, or False if the original request has yet to be SUCCESSFULLY and FULLY addressed)
    - Are we in a loop where we are repeating the same requests and / or getting the same responses as before? Loops can span multiple turns, and can include repeated actions like scrolling up or down more than a handful of times.
    - Are we making forward progress? (True if just starting, or recent messages are adding value. False if recent messages show evidence of being stuck in a loop or if there is evidence of significant barriers to success such as the inability to read from a required file)
    - Who should speak next? (select from: {names})
    - What instruction or question would you give this team member? (Phrase as if speaking directly to them, and include any specific information they may need)

Please output an answer in pure JSON format according to the following schema. The JSON object must be parsable as-is. DO NOT OUTPUT ANYTHING OTHER THAN JSON, AND DO NOT DEVIATE FROM THIS SCHEMA:

    {{
       "is_request_satisfied": {{
            "reason": string,
            "answer": boolean
        }},
        "is_in_loop": {{
            "reason": string,
            "answer": boolean
        }},
        "is_progress_being_made": {{
            "reason": string,
            "answer": boolean
        }},
        "next_speaker": {{
            "reason": string,
            "answer": string (select from: {names})
        }},
        "instruction_or_question": {{
            "reason": string,
            "answer": string
        }}
    }}
"""
```

### ORCHESTRATOR_TASK_LEDGER_FACTS_UPDATE_PROMPT

**Purpose:** Updates the fact sheet when progress stalls. This prompt helps the orchestrator learn from attempts and update its understanding, particularly focusing on educated guesses and moving guesses to verified facts.

**Usage:** Called when the team is not making progress to refresh the fact sheet with new learnings.

**Variables:**
- `{task}`: The user's original request
- `{facts}`: The current fact sheet

```python
ORCHESTRATOR_TASK_LEDGER_FACTS_UPDATE_PROMPT = """As a reminder, we are working to solve the following task:

{task}

It's clear we aren't making as much progress as we would like, but we may have learned something new. Please rewrite the following fact sheet, updating it to include anything new we have learned that may be helpful. Example edits can include (but are not limited to) adding new guesses, moving educated guesses to verified facts if appropriate, etc. Updates may be made to any section of the fact sheet, and more than one section of the fact sheet can be edited. This is an especially good time to update educated guesses, so please at least add or update one educated guess or hunch, and explain your reasoning.

Here is the old fact sheet:

{facts}
"""
```

### ORCHESTRATOR_TASK_LEDGER_PLAN_UPDATE_PROMPT

**Purpose:** Revises the execution plan after failures or when stuck in a loop. This prompt encourages root cause analysis and learning from mistakes to create an improved plan.

**Usage:** Called when the team needs to revise its strategy due to failures or lack of progress.

**Variables:**
- `{team}`: Description of available team members

```python
ORCHESTRATOR_TASK_LEDGER_PLAN_UPDATE_PROMPT = """Please briefly explain what went wrong on this last run (the root cause of the failure), and then come up with a new plan that takes steps and/or includes hints to overcome prior challenges and especially avoids repeating the same mistakes. As before, the new plan should be concise, be expressed in bullet-point form, and consider the following team composition (do not involve any other outside people since we cannot contact anyone else):

{team}
"""
```

### ORCHESTRATOR_FINAL_ANSWER_PROMPT

**Purpose:** Generates the final answer to present to the user after the task is completed. This prompt instructs the AI to synthesize all the information gathered during execution into a clear, user-facing response.

**Usage:** Called after the task is marked as complete to generate the final answer.

**Variables:**
- `{task}`: The user's original request

```python
ORCHESTRATOR_FINAL_ANSWER_PROMPT = """
We are working on the following task:
{task}

We have completed the task.

The above messages contain the conversation that took place to complete the task.

Based on the information gathered, provide the final answer to the original request.
The answer should be phrased as if you were speaking to the user.
"""
```

---

## Web Surfing Prompts

These prompts enable AI agents to navigate and interact with web pages, both in multimodal (with screenshots) and text-only modes.

**Location:** `python/packages/autogen-ext/src/autogen_ext/agents/web_surfer/_prompts.py`

### WEB_SURFER_TOOL_PROMPT_MM

**Purpose:** Multimodal web surfing prompt that enables the AI to interact with webpages using visual information. The prompt describes interactive elements with bounding boxes and numeric IDs overlaid on screenshots, allowing the agent to understand the page layout visually and select appropriate actions.

**Usage:** Used when the web surfer agent operates in multimodal mode with screenshot capabilities.

**Variables:**
- `{state_description}`: Current state of the browser/page
- `{visible_targets}`: List of visible interactive elements with their bounding box IDs
- `{other_targets_str}`: Additional targets not currently visible
- `{focused_hint}`: Information about the currently focused element
- `{tool_names}`: Available tools for interaction
- `{title}`: Current page title
- `{url}`: Current page URL

```python
WEB_SURFER_TOOL_PROMPT_MM = """
{state_description}

Consider the following screenshot of the page. In this screenshot, interactive elements are outlined in bounding boxes of different colors. Each bounding box has a numeric ID label in the same color. Additional information about each visible label is listed below:

{visible_targets}{other_targets_str}{focused_hint}

You are to respond to my next request by selecting an appropriate tool from the following set, or by answering the question directly if possible:

{tool_names}

When deciding between tools, consider if the request can be best addressed by:
    - the contents of the CURRENT VIEWPORT (in which case actions like clicking links, clicking buttons, inputting text, or hovering over an element, might be more appropriate)
    - contents found elsewhere on the CURRENT WEBPAGE [{title}]({url}), in which case actions like scrolling, summarization, or full-page Q&A might be most appropriate
    - on ANOTHER WEBSITE entirely (in which case actions like performing a new web search might be the best option)

My request follows:
"""
```

### WEB_SURFER_TOOL_PROMPT_TEXT

**Purpose:** Text-only version of the web surfing prompt for environments without multimodal capabilities. Instead of visual bounding boxes, it relies on textual descriptions of interactive components.

**Usage:** Used when the web surfer agent operates in text-only mode without screenshot capabilities.

**Variables:**
- `{state_description}`: Current state of the browser/page
- `{visible_targets}`: List of visible interactive elements
- `{other_targets_str}`: Additional targets not currently visible
- `{focused_hint}`: Information about the currently focused element
- `{tool_names}`: Available tools for interaction
- `{title}`: Current page title
- `{url}`: Current page URL

```python
WEB_SURFER_TOOL_PROMPT_TEXT = """
{state_description}

You have also identified the following interactive components:

{visible_targets}{other_targets_str}{focused_hint}

You are to respond to my next request by selecting an appropriate tool from the following set, or by answering the question directly if possible:

{tool_names}

When deciding between tools, consider if the request can be best addressed by:
    - the contents of the CURRENT VIEWPORT (in which case actions like clicking links, clicking buttons, inputting text, or hovering over an element, might be more appropriate)
    - contents found elsewhere on the CURRENT WEBPAGE [{title}]({url}), in which case actions like scrolling, summarization, or full-page Q&A might be most appropriate
    - on ANOTHER WEBSITE entirely (in which case actions like performing a new web search might be the best option)

My request follows:
"""
```

### WEB_SURFER_QA_SYSTEM_MESSAGE

**Purpose:** System message for the document summarization capability of the web surfer. Sets the agent's role as a helpful assistant specialized in summarizing long documents.

**Usage:** Used as the system message when the web surfer needs to summarize webpage content.

```python
WEB_SURFER_QA_SYSTEM_MESSAGE = """
You are a helpful assistant that can summarize long documents to answer question.
"""
```

### WEB_SURFER_QA_PROMPT

**Purpose:** Generates prompts for summarizing webpage content, either with respect to a specific question or as a general summary. This function dynamically creates the appropriate prompt based on whether a question is provided.

**Usage:** Called when the web surfer needs to summarize a webpage's full-text content.

**Parameters:**
- `title` (str): The webpage title
- `question` (str | None): Optional specific question to guide the summary

**Returns:** Formatted prompt string

```python
def WEB_SURFER_QA_PROMPT(title: str, question: str | None = None) -> str:
    base_prompt = f"We are visiting the webpage '{title}'. Its full-text content are pasted below, along with a screenshot of the page's current viewport."
    if question is not None:
        return (
            f"{base_prompt} Please summarize the webpage into one or two paragraphs with respect to '{question}':\n\n"
        )
    else:
        return f"{base_prompt} Please summarize the webpage into one or two paragraphs:\n\n"
```

