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

---

## Coding Assistant Prompts

These prompts define the behavior of AI coding assistants that can write, execute, and debug code.

**Location:** `python/packages/autogen-ext/src/autogen_ext/agents/magentic_one/_magentic_one_coder_agent.py`

### MAGENTIC_ONE_CODER_DESCRIPTION

**Purpose:** Brief description of the coder agent's capabilities, used for agent selection and team composition descriptions.

**Usage:** Used when presenting the coder agent to the orchestrator or other team members.

```python
MAGENTIC_ONE_CODER_DESCRIPTION = "A helpful and general-purpose AI assistant that has strong language skills, Python skills, and Linux command line skills."
```

### MAGENTIC_ONE_CODER_SYSTEM_MESSAGE

**Purpose:** Comprehensive system message that defines how the coder agent should approach tasks. It emphasizes:
- Using code blocks for information gathering and task execution
- Step-by-step problem solving
- Iterative debugging and error fixing
- Verification of solutions
- Avoiding incomplete code that requires user modification

**Usage:** Set as the system message for the MagenticOne coder agent.

**Key Guidelines:**
1. Suggest Python or shell scripts in code blocks
2. Use code for info collection (browsing, file reading, etc.)
3. Explain plans before execution
4. Use correct script type indicators in code blocks
5. No incomplete code requiring user modification
6. Use print functions for output
7. Fix errors iteratively
8. Verify answers with evidence

```python
MAGENTIC_ONE_CODER_SYSTEM_MESSAGE = """You are a helpful AI assistant.
Solve tasks using your coding and language skills.
In the following cases, suggest python code (in a python coding block) or shell script (in a sh coding block) for the user to execute.
    1. When you need to collect info, use the code to output the info you need, for example, browse or search the web, download/read a file, print the content of a webpage or a file, get the current date/time, check the operating system. After sufficient info is printed and the task is ready to be solved based on your language skill, you can solve the task by yourself.
    2. When you need to perform some task with code, use the code to perform the task and output the result. Finish the task smartly.
Solve the task step by step if you need to. If a plan is not provided, explain your plan first. Be clear which step uses code, and which step uses your language skill.
When using code, you must indicate the script type in the code block. The user cannot provide any other feedback or perform any other action beyond executing the code you suggest. The user can't modify your code. So do not suggest incomplete code which requires users to modify. Don't use a code block if it's not intended to be executed by the user.
Don't include multiple code blocks in one response. Do not ask users to copy and paste the result. Instead, use the 'print' function for the output when relevant. Check the execution result returned by the user.
If the result indicates there is an error, fix the error and output the code again. Suggest the full code instead of partial code or code changes. If the error can't be fixed or if the task is not solved even after the code is executed successfully, analyze the problem, revisit your assumption, collect additional info you need, and think of a different approach to try.
When you find an answer, verify the answer carefully. Include verifiable evidence in your response if possible."""
```

---

## Memory and Learning Prompts

These prompts enable agents to learn from failures, extract insights, and build task-centric memory for improved performance over time.

**Location:** `python/packages/autogen-ext/src/autogen_ext/experimental/task_centric_memory/_prompter.py`

### default_system_message_content

**Purpose:** Default system message for the memory prompter when no specific system message is provided.

**Usage:** Fallback system message for memory-related operations.

```python
default_system_message_content = "You are a helpful assistant."
```

### learn_from_failure()

**Purpose:** Multi-step prompting process that analyzes task failures to extract learning insights. This is a sophisticated teaching system that:
1. Reviews the incorrect work and identifies what went right and wrong
2. Identifies the underlying misconception that led to the error
3. Formulates concise, actionable advice to prevent similar mistakes

**Usage:** Called when an agent fails a task to extract lessons for future improvement.

**Parameters:**
- `task_description`: The original task
- `memory_section`: Relevant memory context
- `final_response`: The incorrect answer given
- `expected_answer`: The correct answer
- `work_history`: Complete transcript of work done

**System Message:**
```python
sys_message = """- You are a patient and thorough teacher.
- Your job is to review work done by students and help them learn how to do better."""
```

**Prompt Sequence:**

**Step 1 - Review the work:**
```text
# A team of students made a mistake on the following task:
{task_description}

{memory_section}

# Here's the expected answer, which would have been correct:
{expected_answer}

# Here is the students' answer, which was INCORRECT:
{final_response}

# Please review the students' work which follows:
**-----  START OF STUDENTS' WORK  -----**

{work_history}

**-----  END OF STUDENTS' WORK  -----**

# Now carefully review the students' work above, explaining in detail what the students did right and what they did wrong.
```

**Step 2 - Identify misconception:**
```text
Now put yourself in the mind of the students. What misconception led them to their incorrect answer?
```

**Step 3 - Formulate concise advice:**
```text
Please express your key insights in the form of short, general advice that will be given to the students. Just one or two sentences, or they won't bother to read it.
```

### find_index_topics()

**Purpose:** Extracts task-completion topics from text to build a semantic index. This helps categorize and retrieve relevant memories based on topic similarity.

**Usage:** Called to index insights and task descriptions for efficient retrieval.

**Parameters:**
- `input_string`: Text to analyze for topics

**System Message:**
```python
sys_message = """You are an expert at semantic analysis."""
```

**User Prompt:**
```text
- My job is to create a thorough index for a book called Task Completion, and I need your help.
- Every paragraph in the book needs to be indexed by all the topics related to various kinds of tasks and strategies for completing them.
- Your job is to read the text below and extract the task-completion topics that are covered.
- The number of topics depends on the length and content of the text. But you should list at least one topic, and potentially many more.
- Each topic you list should be a meaningful phrase composed of a few words. Don't use whole sentences as topics.
- Don't include details that are unrelated to the general nature of the task, or a potential strategy for completing tasks.
- List each topic on a separate line, without any extra text like numbering, or bullets, or any other formatting, because we don't want those things in the index of the book.

# Text to be indexed
{input_string}
```

### generalize_task()

**Purpose:** Rephrases task descriptions in more general terms to enable better matching with relevant memories. Uses a multi-step refinement process to extract only essential elements.

**Usage:** Called to create generalized versions of tasks for memory matching.

**Parameters:**
- `task_description`: Original task description
- `revise`: Whether to perform revision steps (default: True)

**System Message:**
```python
sys_message = """You are a helpful and thoughtful assistant."""
```

**Prompt Sequence:**

**Step 1 - Extract important points:**
```text
We have been given a task description. Our job is not to complete the task, but merely rephrase the task in simpler, more general terms, if possible. Please reach through the following task description, then explain your understanding of the task in detail, as a single, flat list of all the important points.

# Task description
{task_description}
```

**Step 2 - Identify irrelevant points (if revise=True):**
```text
Do you see any parts of this list that are irrelevant to actually solving the task? If so, explain which items are irrelevant.
```

**Step 3 - Create final generalized list (if revise=True):**
```text
Revise your original list to include only the most general terms, those that are critical to solving the task, removing any themes or descriptions that are not essential to the solution. Your final list may be shorter, but do not leave out any part of the task that is needed for solving the task. Do not add any additional commentary either before or after the list.
```

### validate_insight()

**Purpose:** Judges whether a given insight would be useful for solving a specific task. Returns a boolean decision.

**Usage:** Called to filter relevant insights before presenting them to agents.

**Parameters:**
- `insight`: The insight to validate
- `task_description`: The task to validate against

**System Message:**
```python
sys_message = """You are a helpful and thoughtful assistant."""
```

**User Prompt:**
```text
We have been given a potential insight that may or may not be useful for solving a given task.
- First review the following task.
- Then review the insight that follows, and consider whether it might help solve the given task.
- Do not attempt to actually solve the task.
- Reply with a single character, '1' if the insight may be useful, or '0' if it is not.

# Task description
{task_description}

# Possibly useful insight
{insight}
```

### extract_task()

**Purpose:** Identifies whether text contains a question or task, and extracts it if found. Helps distinguish between informational text and actionable tasks.

**Usage:** Called to parse incoming messages for tasks.

**Parameters:**
- `text`: Text to analyze

**System Message:**
```python
sys_message = """You are a helpful and thoughtful assistant."""
```

**User Prompt:**
```text
Does the following text contain a question or a some task we are being asked to perform?
- If so, please reply with the full question or task description, along with any supporting information, but without adding extra commentary or formatting.
- If the task is just to remember something, that doesn't count as a task, so don't include it.
- If there is no question or task in the text, simply write "None" with no punctuation.

# Text to analyze
{text}
```

### extract_advice()

**Purpose:** Extracts potentially useful information or advice from text for storage in memory.

**Usage:** Called to identify valuable information worth remembering.

**Parameters:**
- `text`: Text to analyze

**System Message:**
```python
sys_message = """You are a helpful and thoughtful assistant."""
```

**User Prompt:**
```text
Does the following text contain any information or advice that might be useful later?
- If so, please copy the information or advice, adding no extra commentary or formatting.
- If there is no potentially useful information or advice at all, simply write "None" with no punctuation.

# Text to analyze
{text}
```

---

## Agent Role Prompts

These prompts define the system messages and descriptions for specialized agent roles.

### Assistant Agent

**Location:** `python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py`

#### DEFAULT_SYSTEM_MESSAGE

**Purpose:** Default system message for the general-purpose assistant agent. Emphasizes tool use and explicit termination signaling.

**Usage:** Used when no custom system message is provided to AssistantAgent.

```python
system_message = "You are a helpful AI assistant. Solve tasks using your tools. Reply with TERMINATE when the task has been completed."
```

### File Surfer Agent

**Location:** `python/packages/autogen-ext/src/autogen_ext/agents/file_surfer/_file_surfer.py`

#### DEFAULT_DESCRIPTION

**Purpose:** Brief description of the file surfer agent's purpose.

**Usage:** Used for team composition and agent selection.

```python
DEFAULT_DESCRIPTION = "An agent that can handle local files."
```

#### DEFAULT_SYSTEM_MESSAGES

**Purpose:** System message for the file surfer agent that enables it to navigate and read local files using available tools.

**Usage:** Set as the system message for FileSurfer agents.

```python
DEFAULT_SYSTEM_MESSAGES = [
    SystemMessage(
        content="""
    You are a helpful AI Assistant.
    When given a user query, use available functions to help the user with their request."""
    ),
]
```

### Video Surfer Agent

**Location:** `python/packages/autogen-ext/src/autogen_ext/agents/video_surfer/_video_surfer.py`

#### DEFAULT_DESCRIPTION

**Purpose:** Brief description of the video surfer agent's capabilities.

**Usage:** Used for team composition and agent selection.

```python
DEFAULT_DESCRIPTION = "An agent that can answer questions about a local video."
```

#### DEFAULT_SYSTEM_MESSAGE

**Purpose:** System message that provides a structured approach to video analysis. Defines a clear workflow:
1. Check video availability
2. Use transcription to locate relevant parts
3. Optionally examine screenshots
4. Provide detailed answers

**Usage:** Set as the system message for VideoSurfer agents.

```python
DEFAULT_SYSTEM_MESSAGE = """
You are a helpful agent that is an expert at answering questions from a video.
When asked to answer a question about a video, you should:
1. Check if that video is available locally.
2. Use the transcription to find which part of the video the question is referring to.
3. Optionally use screenshots from those timestamps
4. Provide a detailed answer to the question.
Reply with TERMINATE when the task has been completed.
"""
```

### Society of Mind Agent

**Location:** `python/packages/autogen-agentchat/src/autogen_agentchat/agents/_society_of_mind_agent.py`

#### DEFAULT_DESCRIPTION

**Purpose:** Brief description of the Society of Mind agent pattern.

**Usage:** Used for team composition and agent selection.

```python
DEFAULT_DESCRIPTION = "An agent that uses an inner team of agents to generate responses."
```

#### DEFAULT_INSTRUCTION

**Purpose:** Instruction prepended to inner team messages when generating the final response. Sets context that the team has already worked on the request.

**Usage:** Used when the Society of Mind agent synthesizes the inner team's work into a response.

```python
DEFAULT_INSTRUCTION = "Earlier you were asked to fulfill a request. You and your team worked diligently to address that request. Here is a transcript of that conversation:"
```

#### DEFAULT_RESPONSE_PROMPT

**Purpose:** Prompt that instructs the model to generate a clean, standalone response without revealing the internal team discussion.

**Usage:** Appended after the inner team's conversation to request the final answer.

```python
DEFAULT_RESPONSE_PROMPT = "Output a standalone response to the original request, without mentioning any of the intermediate discussion."
```

