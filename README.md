# Eidetic Engine in Plain English:

Agent Loop Code ([agent_master_loop.py](https://github.com/Dicklesworthstone/ultimate_mcp_client/blob/main/agent_master_loop.py)):

Here is plain English, high-level summary of what each method in `agent_master_loop.py` is primarily responsible for. This helps in understanding the overall architecture and flow without getting bogged down in implementation details immediately.


**Class `AgentMasterLoop`**

*   **`__init__(self, mcp_client_instance, default_llm_model_string, agent_state_file)`**
    *   **Purpose:** Sets up the agent when it's first created.
    *   **Does:** It stores essential connections (like the MCPClient to talk to servers/tools), remembers which AI model it should use for its main thinking, and knows where to save/load its progress. It also initializes its internal "state" (like current tasks, plans, error counts) to a fresh start and prepares systems for logging and background tasks.

*   **`async def shutdown(self)`**
    *   **Purpose:** To safely stop the agent and clean up.
    *   **Does:** It signals all parts of the agent to stop, waits for any background activities to finish cleanly, and then saves its current state so it can resume later if needed.

*   **`_construct_agent_prompt(self, current_task_goal_desc, context)`**
    *   **Purpose:** To create the detailed message (prompt) that will be sent to the main AI model (LLM) for decision-making.
    *   **Does:** It assembles a comprehensive set of instructions for the LLM, including its identity, overall process, available tools (with how to use them), and strategies for recovering from errors. It then adds the current situation: the agent's understanding of its goals, its current plan, any recent errors, and relevant information from its memory system (UMS). The prompt is tailored based on whether the agent is starting a brand new task or continuing an existing one.

*   **`_background_task_done(self, task)` and `_background_task_done_safe(self, task)`**
    *   **Purpose:** To handle the completion (or failure) of a background task.
    *   **Does:** When a background activity finishes, these helpers log any errors, remove the task from the active list, and free up resources so another background task can start if needed.

*   **`_start_background_task(self, coro_fn, *args, **kwargs)`**
    *   **Purpose:** To initiate a new background activity without stopping the agent's main work.
    *   **Does:** It takes a function that needs to be run in the background, creates a new managed task for it, and adds it to a list of active background jobs. It ensures these tasks don't run all at once if there are too many.

*   **`_add_bg_task(self, task)`**
    *   **Purpose:** Internal helper to safely add a newly created background task to the agent's tracking set.
    *   **Does:** Adds the provided task object to the `self.state.background_tasks` set, ensuring thread-safety with a lock.

*   **`_cleanup_background_tasks(self)`**
    *   **Purpose:** To stop and clean up all active background tasks, usually during shutdown.
    *   **Does:** It goes through all currently running background activities, cancels them, and waits for them to acknowledge the cancellation, logging any issues.

*   **`async def _estimate_tokens_anthropic(self, data)`**
    *   **Purpose:** A helper to guess how many "pieces" or "tokens" a piece of text will count as for Anthropic's AI models.
    *   **Does:** It sends the text to Anthropic's API (if available) to get an accurate count, or uses a rough character-based estimation if the API call fails. This helps manage how much information is sent to the AI to avoid exceeding limits.

*   **`async def _with_retries(self, coro_fun, ...)`**
    *   **Purpose:** A general helper (decorator pattern) to automatically retry a function if it fails.
    *   **Does:** It wraps another function. If the wrapped function fails with certain types of errors (like network issues or temporary unavailability), it waits a short, increasing amount of time and tries again, up to a set number of retries.

*   **`async def _save_agent_state(self)`**
    *   **Purpose:** To save the agent's current operational state to a file.
    *   **Does:** It gathers all important current information about the agent (like its current task ID, goal list, plan, error counts, etc.) and writes it to a JSON file so the agent can resume later.

*   **`async def _load_agent_state(self)`**
    *   **Purpose:** To load a previously saved operational state from a file.
    *   **Does:** It reads the JSON file, restores the agent's internal information, and performs some checks to ensure the loaded state is valid. If no file exists or it's corrupted, it starts the agent with a fresh, default state.

*   **`_find_tool_server(self, tool_name)`**
    *   **Purpose:** To figure out which connected server computer has the specific tool the agent wants to use.
    *   **Does:** It looks at the list of available tools from all connected MCP servers and returns the name of the server that provides the requested tool.

*   **`async def inject_manual_thought(self, content, thought_type)`**
    *   **Purpose:** To allow an external operator (like a human user) to directly insert a thought or piece of guidance into the agent's ongoing reasoning process.
    *   **Does:** It takes the provided text, records it as a "user guidance" thought in the agent's current UMS thought chain, and also stores it as a distinct, high-priority memory in the UMS. It then flags the agent that it might need to reconsider its current plan based on this new input.

*   **`async def initialize(self)`**
    *   **Purpose:** The main setup routine when the agent first starts or restarts.
    *   **Does:** It loads any saved state, connects to the MCPClient to understand available tools and servers, gets the list of tool schemas formatted for its AI model, checks if all essential tools are available, and then validates its current workflow and goal status with the UMS, potentially setting up a default thought chain or initial UMS goal if resuming an existing workflow.

*   **`async def _set_default_thought_chain_id(self)`**
    *   **Purpose:** To ensure there's an active "notepad" (thought chain) in the UMS for the current task where the agent can record its reasoning.
    *   **Does:** If the agent is working on a task (workflow) but doesn't have a specific thought chain ID assigned, this function queries the UMS for the primary thought chain associated with that workflow and sets it as the agent's current one.

*   **`async def _check_workflow_exists(self, workflow_id)`**
    *   **Purpose:** To verify with the UMS if a given workflow ID is valid and known.
    *   **Does:** It calls a UMS tool (`get_workflow_details`) to check if the workflow exists.

*   **`async def _validate_goal_stack_on_load(self)`**
    *   **Purpose:** When the agent loads its state, this ensures its internal list of current goals (goal stack) is synchronized and consistent with what's actually stored in the UMS.
    *   **Does:** If the agent loaded a `current_goal_id`, it attempts to rebuild the goal hierarchy from that ID upwards by querying UMS. If the UMS version matches or is valid, the agent's local stack is updated. If not, or if no `current_goal_id` was loaded, the local stack is cleared to avoid inconsistency.

*   **`_detect_plan_cycle(self, plan)`**
    *   **Purpose:** To check if the agent's current plan has any circular dependencies (e.g., step A needs B, B needs C, C needs A).
    *   **Does:** It analyzes the `depends_on` fields of the plan steps to see if any step is waiting for another step that, directly or indirectly, is waiting for the first step, which would create an unresolvable loop.

*   **`async def _check_prerequisites(self, ids)`**
    *   **Purpose:** Before executing a plan step that depends on other UMS actions, this checks if those prerequisite actions have been completed in the UMS.
    *   **Does:** It takes a list of UMS action IDs and calls a UMS tool (`get_action_details`) to query their status, returning whether all of them are marked "completed".

*   **`async def _record_action_start_internal(self, tool_name, tool_args, planned_dependencies)`**
    *   **Purpose:** An internal helper to log the beginning of an agent's action (typically a tool call) in the UMS.
    *   **Does:** It calls the UMS tool `record_action_start` with details about the tool being used and its arguments. If the action being started depends on other previously recorded UMS actions (based on the agent's plan), it also calls `_record_action_dependencies_internal`.

*   **`async def _record_action_dependencies_internal(self, source_id, target_ids)`**
    *   **Purpose:** An internal helper to log dependencies between UMS actions.
    *   **Does:** For a given source UMS action ID, it calls the UMS tool `add_action_dependency` for each target UMS action ID it depends on.

*   **`async def _record_action_completion_internal(self, action_id, result)`**
    *   **Purpose:** An internal helper to log the completion (or failure) of an agent's action in the UMS.
    *   **Does:** It calls the UMS tool `record_action_completion` with the UMS action ID, the outcome status (completed/failed), and the result obtained from the tool.

*   **`async def _run_auto_linking(self, memory_id, *, workflow_id, context_id)`**
    *   **Purpose:** A background task to automatically try to find and create semantic links between a newly stored memory and existing memories in the UMS.
    *   **Does:** After a short delay, it takes a new memory, searches the UMS for other semantically similar memories, and if any are found above a certain similarity threshold, it calls the UMS tool `create_memory_link` to create a "related" link between them.

*   **`async def _check_and_trigger_promotion(self, memory_id, *, workflow_id, context_id)`**
    *   **Purpose:** A background task to check if a memory in UMS has become important enough (e.g., due to frequent access or high confidence) to be "promoted" to a higher cognitive level (like from episodic to semantic).
    *   **Does:** It calls the UMS tool `promote_memory_level` for the given memory ID. The UMS tool itself contains the logic to decide if promotion is warranted and performs the update.

*   **`async def _execute_tool_call_internal(self, tool_name, arguments, record_action, planned_dependencies)`**
    *   **Purpose:** The agent's central function for executing any tool, whether it's a UMS tool or an internal agent function.
    *   **Does:** It finds the server for the tool, automatically adds common arguments like `workflow_id` if needed, checks prerequisites (via `_check_prerequisites`), records the action start in UMS (if `record_action` is true and it's not an internal/meta tool), executes the tool via `MCPClient.execute_tool` (with retries), records the action completion in UMS, updates internal tool usage statistics, and triggers background tasks for memory linking or promotion if the tool call resulted in new memories or significant memory access. It also handles errors and updates the agent's `last_error_details`.

*   **`async def _handle_workflow_and_goal_side_effects(self, tool_name, arguments, result_content)`**
    *   **Purpose:** To update the agent's internal state (`self.state`) after specific UMS tools related to workflow or goal management have been successfully executed.
    *   **Does:**
        *   If `TOOL_CREATE_WORKFLOW` succeeded: It sets the agent's current `workflow_id`, `context_id`, `thought_chain_id`, creates the root UMS goal for this new workflow (by calling `TOOL_CREATE_GOAL`), updates the agent's `goal_stack` and `current_goal_id` to this new root UMS goal, and sets an initial plan step to assess this new UMS goal.
        *   If `TOOL_CREATE_GOAL` (called by the LLM) succeeded: It adds the new UMS goal to the agent's `goal_stack`, makes it the `current_goal_id`, and sets a plan to start working on this new sub-goal.
        *   If `TOOL_UPDATE_GOAL_STATUS` succeeded (marking a UMS goal as completed/failed): It updates the status of that goal in the agent's local `goal_stack`. If the marked goal was the agent's `current_goal_id` and is now terminal, it pops it, sets the `current_goal_id` to its parent (from UMS result), rebuilds the local `goal_stack` view from UMS for this new parent, and sets a plan to re-assess. If a root UMS goal was finished, it sets `self.state.goal_achieved_flag`.
        *   If `TOOL_UPDATE_WORKFLOW_STATUS` succeeded (marking a workflow terminal): If it was the agent's current workflow, it pops it from the `workflow_stack`. If there's a parent workflow, it switches focus to that; otherwise, it signals overall completion.

*   **`async def _fetch_goal_stack_from_ums(self, leaf_goal_id)`**
    *   **Purpose:** To get the full hierarchy of parent goals from the UMS, leading up to a given (leaf) goal ID.
    *   **Does:** Starting with the `leaf_goal_id`, it repeatedly calls the UMS tool `get_goal_details` to fetch a goal, then gets its `parent_goal_id`, and fetches that parent, continuing until it reaches a goal with no parent (a root goal) or hits a depth limit. It returns these goals as a list, ordered from root to leaf.

*   **`async def _apply_heuristic_plan_update(self, last_decision, last_tool_result_content)`**
    *   **Purpose:** A fallback mechanism to update the agent's plan if the LLM doesn't explicitly provide a new plan (e.g., after a simple thought or a tool call that doesn't inherently lead to a replan).
    *   **Does:** It looks at the last LLM decision and tool result. If an action was successful, it marks the current plan step as "completed," adds a summary of the result, and removes the step from the plan. If the plan becomes empty, it adds a default next step (like "Analyze overall result"). If an action failed, it marks the step "failed" and inserts a new step to "Analyze failure... and replan," setting `needs_replan` to true. It also manages error counters and success counters for meta-cognition.

*   **`_adapt_thresholds(self, stats)`**
    *   **Purpose:** To dynamically adjust how frequently the agent performs self-reflection and memory consolidation based on its recent performance and memory state.
    *   **Does:** It looks at UMS statistics (like the ratio of temporary vs. permanent memories, tool failure rates). If there are too many temporary memories, it makes the agent consolidate memories sooner. If there are many errors, it makes the agent reflect on its process sooner. If things are going smoothly, it lets the agent work longer before these meta-cognitive pauses.

*   **`async def _run_periodic_tasks(self)`**
    *   **Purpose:** The agent's internal "scheduler" for triggering various background and meta-cognitive activities.
    *   **Does:** It checks several counters/timers. If enough successful actions have occurred or enough loops have passed, it triggers UMS tools for:
        *   Reflection (`generate_reflection`) on its recent activities.
        *   Memory consolidation (`consolidate_memories`) to summarize or find insights in existing memories.
        *   Working memory optimization (`optimize_working_memory`) and focus update (`auto_update_focus`).
        *   Checking if any memories are ready for promotion (`_trigger_promotion_checks`).
        *   Fetching UMS statistics (`compute_memory_statistics`) to adapt its thresholds.
        *   Running UMS maintenance (`delete_expired_memories`).

*   **`async def _trigger_promotion_checks(self)`**
    *   **Purpose:** To identify candidate memories in UMS that might be ready for promotion to a higher cognitive level and start background checks for them.
    *   **Does:** It queries the UMS for recently accessed episodic memories and semantic memories that are procedures/skills. For each candidate, it starts a background task (`_check_and_trigger_promotion`) to evaluate if it should be promoted.

*   **`async def _gather_context(self)`** (As per your latest version)
    *   **Purpose:** To collect all the necessary information from the agent's internal state and the UMS that the LLM will need to make its next decision.
    *   **Does:**
        1.  Assembles information from the agent's own state (current plan, last action, errors, workflow stack, current thought chain ID, meta-feedback).
        2.  Constructs the `agent_assembled_goal_context` by:
            *   If a `current_goal_id` is set, it first tries to fetch the full goal stack from UMS leading to this goal using `_fetch_goal_stack_from_ums`.
            *   If the UMS fetch is successful and consistent, it uses that.
            *   If the UMS fetch fails or is inconsistent, but the agent's internal `self.state.goal_stack` seems valid for the `current_goal_id` (important for the turn right after a new root goal is created by the agent), it uses the agent's local stack.
            *   Otherwise, it records an error if goal context can't be determined.
        3.  Calls the UMS tool `get_rich_context_package` to get a bundle of relevant memories (working, core, proactive, procedural) and contextual links from the UMS for the current workflow/context.
        4.  Optionally (though currently commented out in your code), it could perform an agent-side compression of the gathered context if it's too large.
        5.  Returns a comprehensive dictionary (`context_payload`) containing all this information.

*   **`async def prepare_next_turn_data(self, overall_goal)`**
    *   **Purpose:** This is the main method called by the MCPClient to get everything needed for the LLM to take its next "turn."
    *   **Does:** It first runs any scheduled periodic/background tasks (`_run_periodic_tasks`). Then, it gathers all the current context using `_gather_context`. Finally, it uses this context and the `overall_goal` (or current operational goal) to build the actual prompt messages for the LLM via `_construct_agent_prompt`. It returns these messages, the list of tool schemas the LLM can use, and a snapshot of the context that was gathered.

*   **`async def execute_llm_decision(self, llm_decision)`**
    *   **Purpose:** To take the decision made by the LLM and carry it out.
    *   **Does:**
        *   It parses the LLM's decision, which could be:
            *   To call a tool: It then uses `_execute_tool_call_internal` to run the tool.
            *   To record a thought: It calls `TOOL_RECORD_THOUGHT`.
            *   To signal overall completion: It marks the root UMS goal as completed and sets a flag.
            *   To update the plan: It validates and applies the new plan from the LLM.
            *   An error from the LLM: It logs this and sets up for replanning.
        *   If the LLM didn't provide an explicit plan update (and didn't just call the plan update tool), it uses `_apply_heuristic_plan_update` to make a common-sense adjustment to the current plan (e.g., mark step complete, handle failure).
        *   It checks if the agent has made too many consecutive errors and stops if so.
        *   It saves the agent's state.
        *   It determines if the agent should continue looping or stop (e.g., if the goal is achieved, max errors, or shutdown signaled).

*   **`async def run_main_loop(self, initial_goal, max_loops)`**
    *   **Purpose:** This method is called by MCPClient's `run_self_driving_agent` to execute one cycle of the agent's operation when it's running autonomously.
    *   **Does:**
        *   If it's the very first run for this agent instance (no `workflow_id` yet), it calls `TOOL_CREATE_WORKFLOW` (via `_execute_tool_call_internal`) to establish the initial UMS workflow and its root UMS goal based on the `initial_goal`.
        *   It increments its internal loop counter.
        *   It then calls `prepare_next_turn_data` to get the prompt, tools, and context for the LLM.
        *   It returns this package of data to the MCPClient, which will then make the actual LLM call and pass the decision back to the agent's `execute_llm_decision` method (orchestrated by `run_self_driving_agent` in MCPClient).
        *   It signals to MCPClient if the agent believes it should stop (e.g., goal achieved, shutdown).

This should give a good overview of each method's role in the agent's lifecycle!

---

Unified Memory System Code ([unified_memory_system.py](https://github.com/Dicklesworthstone/ultimate_mcp_server/blob/main/ultimate_mcp_server/tools/unified_memory_system.py))

**Overall Purpose:** This module acts as the "brain's librarian and project manager" for your AI agent. It provides a structured way to store, retrieve, and relate all kinds of information (memories, thoughts, actions, created files/artifacts, goals) and to track the progress of tasks (workflows). It uses a database (SQLite) to keep everything organized and persistent.

---

**Key Classes & Their Purpose:**

*   **Enums (e.g., `WorkflowStatus`, `ActionStatus`, `ActionType`, `ArtifactType`, `ThoughtType`, `MemoryLevel`, `MemoryType`, `LinkType`, `GoalStatus`)**
    *   **Purpose:** To define a fixed set of allowed values for different categories.
    *   **Does:** They provide standardized labels (like "active", "completed", "file", "goal", "episodic", "related") to ensure consistency when recording and querying information. For example, an action can only have a status like "planned", "completed", or "failed", not arbitrary text.

*   **`DBConnection` Class**
    *   **Purpose:** To manage the connection to the SQLite database. It's designed to be a "singleton," meaning there's only one active connection instance for the whole system, which improves performance and consistency.
    *   **Does:**
        *   When first used, it connects to the database file (creating it if it doesn't exist).
        *   It sets up the database structure (tables like `workflows`, `actions`, `memories`, `thoughts`, `goals`, etc.) using the `SCHEMA_STATEMENTS` if the database is new.
        *   It configures the database for better performance and reliability (e.g., using WAL mode, enabling foreign keys).
        *   It provides a way to run database operations within a "transaction," which means a group of changes are either all applied successfully or none are (ensuring data integrity).
        *   Handles ensuring only one connection is established and reused.
        *   Provides a method to explicitly close the database connection when the server shuts down.

*   **`MemoryUtils` Class**
    *   **Purpose:** A collection of helper functions for common tasks related to memory data.
    *   **Does:**
        *   `generate_id()`: Creates unique IDs (UUIDs) for new records.
        *   `serialize()`: Safely converts Python objects (like dictionaries or lists for metadata) into a text format (JSON string) that can be stored in the database. It handles cases where objects are complex or too large.
        *   `deserialize()`: Converts JSON strings from the database back into Python objects.
        *   `_validate_sql_identifier()`: (Internal helper) Checks if a given string is safe to use as a table or column name in an SQL query to prevent security issues.
        *   `get_next_sequence_number()`: Finds the next available number in a sequence for ordering items (e.g., actions in a workflow, thoughts in a chain).
        *   `process_tags()`: Manages tags. It ensures any given tags exist in a central `tags` table and then links them to an entity (like a workflow or action) in a separate linking table.
        *   `_log_memory_operation()`: (Internal helper) Records a log entry every time a significant memory-related operation occurs (like creating a memory, accessing it, deleting it).
        *   `_update_memory_access()`: (Internal helper) Updates a memory's "last accessed" timestamp and increments its "access count" whenever it's retrieved.

---

**Key Standalone Functions (Tools exposed by the UMS):**

*   **`_json_contains`, `_json_contains_any`, `_json_contains_all` (SQLite Custom Functions)**
    *   **Purpose:** These are special functions added directly to SQLite to allow searching within JSON data stored in text columns (e.g., searching for specific tags in a JSON list of tags).
    *   **Does:** They enable more powerful queries on JSON-formatted metadata or tags directly within the database.

*   **`_compute_memory_relevance` (SQLite Custom Function)**
    *   **Purpose:** Calculates a "relevance score" for a memory based on several factors.
    *   **Does:** It takes a memory's importance, confidence, creation date, access count, and last access time to compute a score indicating how relevant that memory might be *right now*. This helps in prioritizing which memories to retrieve or show.

*   **`to_iso_z`, `safe_format_timestamp` (Utility Functions)**
    *   **Purpose:** To handle and format timestamps consistently.
    *   **Does:** `to_iso_z` converts a Unix timestamp (a number) into a standard human-readable ISO 8601 date-time string (like "2023-10-27T10:30:00Z"). `safe_format_timestamp` attempts to convert various input timestamp formats (numbers, potentially existing ISO strings) into this consistent "Z" format, handling errors gracefully.

*   **`_store_embedding(conn, memory_id, text)` (Internal Embedding Helper)**
    *   **Purpose:** To generate a "vector embedding" (a list of numbers representing the meaning) for a piece of text associated with a memory and store it.
    *   **Does:** It uses an external "embedding service" (like one from OpenAI or a local model) to convert the text into a vector. This vector is then saved in the `embeddings` table, linked to the `memory_id`. The memory record itself is updated to point to this new embedding entry.

*   **`_find_similar_memories(conn, query_text, ...)` (Internal Semantic Search Helper)**
    *   **Purpose:** To find memories that are semantically similar (i.e., have similar meaning) to a given query text.
    *   **Does:**
        1.  Generates an embedding for the `query_text`.
        2.  Searches the `embeddings` table for stored memory embeddings that are close to the query embedding (using cosine similarity).
        3.  Filters these results based on criteria like workflow, memory level/type, and a similarity threshold.
        4.  Returns a list of (memory_id, similarity_score) for the most similar memories.

*   **`initialize_memory_system(db_path)`**
    *   **Purpose:** (Agent Internal) To set up and verify the entire memory system when the agent or server starts.
    *   **Does:** It ensures the database connection can be established (using `DBConnection`), that the schema is correctly set up, and that the embedding service is functional. It's a startup check.

*   **`create_workflow(title, ...)`**
    *   **Purpose:** To start a new overarching task or project for the agent.
    *   **Does:** Creates a new record in the `workflows` table. If a `goal` description is provided, it also creates an initial UMS goal (via `create_goal`) and a primary "thought chain" where the agent can record its reasoning for this workflow. Returns the ID of the new workflow and its main thought chain.

*   **`update_workflow_status(workflow_id, status, ...)`**
    *   **Purpose:** To change the overall status of a workflow (e.g., mark it as "completed" or "failed").
    *   **Does:** Updates the `status` field (and `completed_at` timestamp if applicable) for the specified workflow in the `workflows` table. It can also add a final thought (like a completion message) to the workflow's main thought chain.

*   **`record_action_start(workflow_id, action_type, reasoning, ...)`**
    *   **Purpose:** To log the beginning of a specific step or action the agent is taking.
    *   **Does:** Creates a new record in the `actions` table. It includes why the action is being taken (`reasoning`), what tool is being used (if any) and its arguments. It sets the action's status to "in_progress". Importantly, it also creates a linked "episodic memory" detailing that this action has started.

*   **`record_action_completion(action_id, status, ...)`**
    *   **Purpose:** To log the end of an agent action, its outcome, and any results.
    *   **Does:** Updates the specified action in the `actions` table with its final `status` (e.g., "completed", "failed"), completion time, and the `tool_result`. It also updates the linked episodic memory about this action to include the completion details. Optionally, it can record a "conclusion thought" related to this action.

*   **`get_action_details(action_id, ...)`**
    *   **Purpose:** To retrieve all stored information about one or more specific actions.
    *   **Does:** Queries the `actions` table (and related tag tables) for the given action ID(s) and returns all their details, optionally including their dependencies.

*   **`summarize_context_block(text_to_summarize, ...)`**
    *   **Purpose:** (Agent Internal Utility) To use an LLM to create a shorter summary of a large block of text.
    *   **Does:** It takes the input text, sends it to an LLM (via the `ultimate_mcp_server.core.providers` system) with a prompt asking for a concise summary, and returns the summary. This is intended for internal UMS use if it needs to compress large pieces of context, not directly by the agent's main LLM.

*   **`add_action_dependency(source_action_id, target_action_id, ...)`**
    *   **Purpose:** To formally state that one action cannot start until another action is finished.
    *   **Does:** Creates a record in the `dependencies` table linking a `source_action_id` (the one that depends) to a `target_action_id` (the prerequisite).

*   **`get_action_dependencies(action_id, direction, ...)`**
    *   **Purpose:** To find out which actions depend on a given action, or which actions a given action depends on.
    *   **Does:** Queries the `dependencies` table to list actions linked to the given `action_id`, based on the specified `direction` ("upstream" for prerequisites, "downstream" for dependents).

*   **`record_artifact(workflow_id, name, artifact_type, ...)`**
    *   **Purpose:** To log a file, piece of data, code snippet, or any other output generated by the agent.
    *   **Does:** Creates a new record in the `artifacts` table. It stores the artifact's name, type, description, and optionally its file path or even a snippet of its content directly in the database (if small). It also creates a linked "episodic memory" about the creation of this artifact.

*   **`record_thought(workflow_id, content, thought_type, ...)`**
    *   **Purpose:** To save a single step in the agent's reasoning process, a goal, a decision, or a reflection.
    *   **Does:** Creates a new record in the `thoughts` table, linking it to a specific `thought_chain_id` (either provided or the default for the workflow). It stores the `content` of the thought and its `thought_type` (e.g., "goal", "question", "decision"). If the thought is deemed important (like a goal or decision), it also creates a linked "semantic memory" entry. Can be called within an existing database transaction if a `conn` is passed.

*   **`store_memory(workflow_id, content, memory_type, ...)`**
    *   **Purpose:** To add a new piece of knowledge, a fact, an observation, or any distinct information unit to the agent's memory. This is the primary way general information gets into UMS.
    *   **Does:** Creates a new record in the `memories` table. It stores the `content`, `memory_type` (e.g., "fact", "observation"), `memory_level` (e.g., "episodic", "semantic"), importance, confidence, and other metadata like tags, source, and reasoning. If requested, it generates and stores a vector embedding for the memory's content (for semantic search) and can suggest links to other similar memories.

*   **`get_memory_by_id(memory_id, ...)`**
    *   **Purpose:** To retrieve a specific memory entry using its unique ID.
    *   **Does:** Queries the `memories` table for the given `memory_id`. It also updates the memory's access statistics (last accessed time, access count). Optionally, it can also fetch directly linked memories (both incoming and outgoing) and a "semantic context" (other memories similar in meaning).

*   **`search_semantic_memories(query, ...)`**
    *   **Purpose:** To find memories whose content is semantically similar to a given query string.
    *   **Does:** Uses the internal `_find_similar_memories` helper (which leverages vector embeddings) to get a list of (memory_id, similarity_score). It then fetches the full details for these matching memories and returns them. Updates access stats for retrieved memories.

*   **`hybrid_search_memories(query, ...)`**
    *   **Purpose:** To search for memories using a combination of keyword search (Full-Text Search or FTS) and semantic similarity, along with various filters.
    *   **Does:**
        1.  Performs a semantic search (like `search_semantic_memories`) to get a set of candidates and their similarity scores.
        2.  Performs a keyword/filtered search using FTS on memory content/description and applies other filters (level, type, importance, etc.) to get another set of candidates and their relevance scores (calculated by `_compute_memory_relevance`).
        3.  Combines the scores from both searches using provided weights (e.g., 60% semantic, 40% keyword).
        4.  Returns a ranked list of the top matching memories with their hybrid scores. Updates access stats for retrieved memories.

*   **`create_memory_link(source_memory_id, target_memory_id, link_type, ...)`**
    *   **Purpose:** To explicitly create a typed connection (e.g., "related", "supports", "contradicts") between two existing memories.
    *   **Does:** Creates a new record in the `memory_links` table, storing the source, target, type, and strength of the link.

*   **`query_memories(workflow_id, memory_level, ...)`**
    *   **Purpose:** To retrieve memories based on a wide range of structured filters (not primarily semantic similarity, but rather metadata like level, type, tags, importance, creation date, etc.) and optional keyword search on content.
    *   **Does:** Builds a complex SQL query based on the provided filters, sorts the results as requested, and returns a paginated list of matching memory objects. It also updates access statistics for the retrieved memories and can optionally include details of their direct links.

*   **`list_workflows(status, tag, ...)`**
    *   **Purpose:** To get a list of existing workflows, with filters.
    *   **Does:** Queries the `workflows` table based on status, tags, and date ranges, returning a paginated list of workflow summaries (ID, title, status, etc.).

*   **`get_workflow_details(workflow_id, ...)`**
    *   **Purpose:** To retrieve all information associated with a specific workflow.
    *   **Does:** Fetches the main workflow record and, if requested, all its associated actions, artifacts, thought chains (with their thoughts), and a sample of its memories.

*   **`get_recent_actions(workflow_id, limit, ...)`**
    *   **Purpose:** To get a list of the most recent actions performed within a specific workflow.
    *   **Does:** Queries the `actions` table for the given `workflow_id`, orders them by sequence number (most recent first), and returns a limited list of action objects.

*   **`get_artifacts(workflow_id, artifact_type, ...)`**
    *   **Purpose:** To list artifacts associated with a workflow, with filters.
    *   **Does:** Queries the `artifacts` table for a given `workflow_id`, applying filters for type, tags, or whether it's a final output, and returns a limited list.

*   **`get_artifact_by_id(artifact_id, ...)`**
    *   **Purpose:** To retrieve details for a specific artifact by its ID.
    *   **Does:** Queries the `artifacts` table for the given ID and returns its full details.

*   **`create_goal(workflow_id, description, ...)`**
    *   **Purpose:** To define a new goal or sub-goal within a workflow in the UMS.
    *   **Does:** Creates a new record in the `goals` table, linking it to the workflow and optionally to a parent goal. Stores description, priority, status, etc. Returns the full created UMS goal object.

*   **`update_goal_status(goal_id, status, ...)`**
    *   **Purpose:** To change the status of a UMS goal (e.g., mark it active, completed, failed).
    *   **Does:** Updates the `status` (and `completed_at` if terminal) for the specified goal in the `goals` table. Returns details of the updated goal, its parent's ID, and a flag indicating if a root goal was just finished.

*   **`get_goal_details(goal_id, ...)`**
    *   **Purpose:** To retrieve all stored information about a specific UMS goal.
    *   **Does:** Queries the `goals` table for the given `goal_id` and returns its full details.

*   **`create_thought_chain(workflow_id, title, ...)`**
    *   **Purpose:** To start a new, distinct line of reasoning or sub-problem analysis, represented as a "chain" of thoughts.
    *   **Does:** Creates a new record in the `thought_chains` table, linked to the workflow. If an `initial_thought` is provided, it also calls `record_thought` to add that as the first thought in the new chain.

*   **`get_thought_chain(thought_chain_id, ...)`**
    *   **Purpose:** To retrieve a specific thought chain and all the thoughts it contains.
    *   **Does:** Fetches the thought chain record and then all associated thought records, ordered by their sequence.

*   **`_add_to_active_memories(conn, context_id, memory_id)` (Internal Working Memory Helper)**
    *   **Purpose:** To add a memory ID to the working memory list of a specific cognitive state, enforcing size limits.
    *   **Does:** Fetches the current working memory list for the given `context_id` (from `cognitive_states` table). If the list is full, it removes the least relevant memory (based on `_compute_memory_relevance`) before adding the new one. Updates the `cognitive_states` record.

*   **`get_working_memory(context_id, ...)`**
    *   **Purpose:** To retrieve the current set of memories that are considered "active" or "in focus" for a given cognitive context (e.g., for the current workflow).
    *   **Does:** Fetches the `cognitive_states` record for the `context_id`, gets the list of `working_memory` IDs, and then retrieves the full details for each of those memories from the `memories` table. Also returns the `focal_memory_id` for that context.

*   **`focus_memory(memory_id, context_id, ...)`**
    *   **Purpose:** To explicitly set a particular memory as the primary item of focus for a cognitive context.
    *   **Does:** Updates the `focal_memory_id` field in the `cognitive_states` table for the given `context_id`. If requested, it also adds this memory to the working memory list for that context (using `_add_to_active_memories`).

*   **`optimize_working_memory(context_id, target_size, strategy, ...)`**
    *   **Purpose:** (Agent Internal) To calculate an optimized set of memories to keep in working memory for a given context, based on different strategies (like relevance, importance, recency, or diversity). This function *calculates* the list but does *not* update the state itself.
    *   **Does:** Fetches the current working memories for the `context_id`. Scores each memory based on the chosen `strategy`. Selects the top `target_size` memories to retain and identifies which ones would be removed. Returns these two lists of memory IDs.

*   **`save_cognitive_state(workflow_id, title, ...)`**
    *   **Purpose:** (Agent Internal) To take a snapshot of the agent's current "mental state" and save it.
    *   **Does:** Creates a new record in the `cognitive_states` table, storing lists of IDs representing the agent's current working memory, focus areas (also memory IDs), relevant recent actions, and current goals (thought IDs). It marks previously saved states for the workflow as no longer being the "latest."

*   **`load_cognitive_state(workflow_id, state_id, ...)`**
    *   **Purpose:** (Agent Internal) To restore a previously saved cognitive state.
    *   **Does:** Fetches a specific state record (or the latest one if no `state_id` is given) from `cognitive_states` and returns the deserialized lists of IDs for working memory, focus areas, actions, and goals.

*   **`get_rich_context_package(workflow_id, ...)`**
    *   **Purpose:** (Agent Core Internal) This is a high-level function designed to be called by the agent loop to get a comprehensive bundle of context information from the UMS for the LLM.
    *   **Does:** It acts as an orchestrator, internally calling several other UMS tools:
        *   `get_workflow_context` (which itself calls `load_cognitive_state`, `get_recent_actions`, `query_memories` for important memories, and fetches key thoughts).
        *   `get_working_memory` (if `context_id` is provided and working memory is requested).
        *   `hybrid_search_memories` to find proactive memories relevant to a `current_plan_step_description`.
        *   `hybrid_search_memories` (filtered for procedural level) to find relevant procedures.
        *   `get_linked_memories` for a focal memory ID.
        *   Optionally, if the assembled package is too large (exceeds a token threshold), it can call `summarize_text` (another UMS tool) to compress parts of the package.
        *   It bundles all this information into a structured dictionary for the agent to use.

*   **`_calculate_focus_score(memory, recent_action_ids, now_unix)` (Internal Helper)**
    *   **Purpose:** A helper function to score a memory based on its attributes and relationship to recent actions, specifically for deciding which memory should be the "focus."
    *   **Does:** Combines the memory's general relevance score (from `_compute_memory_relevance`) with boosts if it's linked to recent actions or is of a type that often indicates current context (like a question or plan).

*   **`auto_update_focus(context_id, ...)`**
    *   **Purpose:** (Agent Internal) To automatically determine and set the most relevant memory in the current working set as the "focal memory" for a cognitive context.
    *   **Does:** It gets all memories in the working set for the `context_id`. It scores each one using `_calculate_focus_score`. If the highest-scoring memory is different from the current focal memory, it updates the `focal_memory_id` in the `cognitive_states` table.

*   **`promote_memory_level(memory_id, target_level, ...)`**
    *   **Purpose:** (Agent Internal) To attempt to elevate a memory's cognitive level (e.g., from "episodic" to "semantic," or "semantic" to "procedural") if it meets certain criteria (like high access count and confidence).
    *   **Does:** Fetches the memory's current details. Checks if it's eligible for promotion to the `target_level` (or the next logical level if none is specified) based on its current level, type, access count, and confidence against configurable thresholds. If eligible, it updates the `memory_level` in the `memories` table.

*   **`update_memory(memory_id, content, ...)`**
    *   **Purpose:** To modify fields of an existing memory (e.g., correct its content, change its importance, add tags).
    *   **Does:** Updates the specified fields (content, importance, tags, etc.) for the given `memory_id` in the `memories` table. If `regenerate_embedding` is true or if content/description changed, it will re-calculate and update the memory's vector embedding.

*   **`get_linked_memories(memory_id, direction, ...)`**
    *   **Purpose:** To find all memories directly linked to a given memory.
    *   **Does:** Queries the `memory_links` table to find all links where the given `memory_id` is either the source or the target (or both, depending on `direction`). It can optionally fetch and include details of these linked memories.

*   **`_generate_consolidation_prompt(memories, consolidation_type)` (Internal Helper)**
    *   **Purpose:** To create a detailed prompt for an LLM to perform a specific type of memory consolidation (e.g., summarize, find insights, derive a procedure).
    *   **Does:** Takes a list of memories and formats them into a text block. Then, based on the `consolidation_type`, it appends specific instructions to guide the LLM on how to process these memories to produce the desired consolidated output.

*   **`_generate_reflection_prompt(workflow_name, operations, memories, reflection_type)` (Internal Helper)**
    *   **Purpose:** To create a detailed prompt for an LLM to perform a reflection on the agent's recent activities.
    *   **Does:** Takes workflow information and a list of recent memory operations (and details of memories referenced in those operations). Based on the `reflection_type` (e.g., summarize progress, identify gaps, plan next steps), it constructs a prompt asking the LLM to analyze these operations and produce the reflection.

*   **`consolidate_memories(workflow_id, target_memories, ...)`**
    *   **Purpose:** (Meta-cognition) To use an LLM to synthesize multiple UMS memories into a new, more abstract memory (like a summary, insight, or procedure).
    *   **Does:**
        1.  Selects source memories either by explicit IDs (`target_memories`) or by a `query_filter`.
        2.  Generates a prompt (using `_generate_consolidation_prompt`) instructing an LLM to perform the specified `consolidation_type`.
        3.  Calls the LLM to get the consolidated content.
        4.  If `store_result` is true, it stores this new content as a new memory in UMS (using `store_memory`), often at a higher `memory_level` (e.g., semantic), and links it back to the source memories.

*   **`generate_reflection(workflow_id, reflection_type, ...)`**
    *   **Purpose:** (Meta-cognition) To use an LLM to analyze the agent's recent UMS operations and generate a reflection (e.g., about progress, knowledge gaps, or planning).
    *   **Does:**
        1.  Fetches recent operations from the `memory_operations` table for the workflow.
        2.  Fetches details of memories referenced in those operations.
        3.  Generates a prompt (using `_generate_reflection_prompt`) for the LLM.
        4.  Calls the LLM to get the reflection content.
        5.  Stores this reflection in the `reflections` table.

*   **`summarize_text(text_to_summarize, ...)`**
    *   **Purpose:** A general-purpose tool to get an LLM-generated summary of any given text.
    *   **Does:** Sends the `text_to_summarize` to an LLM with a summarization prompt. If `record_summary` and `workflow_id` are provided, it stores the resulting summary as a new memory in UMS.

*   **`delete_expired_memories(db_path)`**
    *   **Purpose:** (Agent Internal / Maintenance) To remove memories that have passed their Time-To-Live (TTL).
    *   **Does:** Queries the `memories` table for entries where `ttl > 0` and `created_at + ttl < current_time`, and deletes them.

*   **`compute_memory_statistics(workflow_id, ...)`**
    *   **Purpose:** (Agent Internal) To calculate various statistics about the memory store, either for a specific workflow or globally.
    *   **Does:** Performs several `COUNT` and `AVG` queries on tables like `memories`, `memory_links`, and `tags` to get counts by level/type, average importance/confidence, link counts, etc.

*   **`_mermaid_escape(text)` (Internal Helper)**
    *   **Purpose:** To clean up text so it can be safely used as labels in Mermaid diagrams.
    *   **Does:** Replaces characters that have special meaning in Mermaid (like quotes, brackets, newlines) with their HTML entity equivalents or `<br>`.

*   **`_generate_mermaid_diagram(workflow)` (Internal Helper)**
    *   **Purpose:** To create the text-based diagram definition (in Mermaid syntax) for a workflow's actions and artifacts.
    *   **Does:** It takes the workflow data (including its actions and artifacts), and constructs a Mermaid flowchart definition, creating nodes for the workflow, each action, and each artifact, and then drawing links between them (e.g., action -> creates -> artifact, or parent_action -> child_action).

*   **`_generate_thought_chain_mermaid(thought_chain)` (Internal Helper)**
    *   **Purpose:** To create the text-based diagram definition (in Mermaid syntax) for a thought chain.
    *   **Does:** Takes thought chain data, creates nodes for each thought (styled by thought type), and links them based on parent-child relationships and to any relevant actions/artifacts/memories.

*   **`generate_workflow_report(workflow_id, report_format, ...)`**
    *   **Purpose:** To create a human-readable report of a workflow's activities.
    *   **Does:**
        1.  Fetches full details of the workflow (using `get_workflow_details`).
        2.  Based on the `report_format`:
            *   `markdown` or `html`: Calls internal helper functions (`_generate_professional_report`, `_generate_concise_report`, etc.) to create a Markdown string, then converts to HTML if needed.
            *   `json`: Dumps the fetched workflow data as a JSON string.
            *   `mermaid`: Calls `_generate_mermaid_diagram` to get the Mermaid diagram string.
        3.  Returns the generated report content.

*   **`visualize_reasoning_chain(thought_chain_id, output_format, ...)`**
    *   **Purpose:** To generate a visual representation (Mermaid diagram or JSON tree) of a specific thought chain.
    *   **Does:** Fetches the thought chain data. If format is `mermaid`, calls `_generate_thought_chain_mermaid`. If `json`, it restructures the flat list of thoughts into a hierarchical tree.

*   **`visualize_memory_network(workflow_id, center_memory_id, ...)`**
    *   **Purpose:** To generate a diagram showing how memories are linked together, optionally centered around a specific memory.
    *   **Does:**
        1.  Selects a set of memory IDs to visualize: either top memories from a `workflow_id` or memories within a certain `depth` (number of links away) from a `center_memory_id`.
        2.  Applies filters for memory level or type.
        3.  Fetches details for these selected memories.
        4.  Fetches all the links *between* these selected memories.
        5.  Calls `_generate_memory_network_mermaid` to create the diagram definition.

*   **Report Generation Helpers (`_generate_professional_report`, `_generate_concise_report`, `_generate_narrative_report`, `_generate_technical_report`)**
    *   **Purpose:** These are internal functions called by `generate_workflow_report` to create the actual Markdown content for different report styles.
    *   **Does:** Each function takes the full workflow data and formats it into a Markdown string according to its specific style (e.g., formal sections for professional, brief summaries for concise, story-like for narrative, data-dumps for technical). They use `safe_format_timestamp` for dates and may include details from actions, thoughts, and artifacts based on the `include_details` flag passed to them.

*   **`_generate_memory_network_mermaid(memories, links, center_memory_id)` (Internal Helper)**
    *   **Purpose:** To create the Mermaid diagram syntax for a network of memories.
    *   **Does:** Takes a list of memory data objects and a list of link objects. It defines Mermaid nodes for each memory (styled by level, highlighting the `center_memory_id`) and then draws edges between them based on the `links` data.

