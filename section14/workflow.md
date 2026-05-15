# Orchestrator-Worker Workflow Explanation

## Overview

This workflow implements a dynamic orchestrator-worker pattern where a central LLM breaks down complex tasks into subtasks, delegates them to worker LLMs running in parallel, and synthesizes their results into a final output. The key characteristic is that subtasks are determined dynamically by the orchestrator based on the input, not pre-defined.

---

## Visual Workflow Graph

```
                                    START
                                      |
                                      v
                            +-------------------+
                            |   orchestrator()  |
                            | (Planning Phase)  |
                            +-------------------+
                                      |
                                      v
                            +-------------------+
                            | assign_workers()  |
                            | (Conditional Edge)|
                            +-------------------+
                                      |
                    +-----------------+-----------------+
                    |                 |                 |
                    v                 v                 v
            +--------------+  +--------------+  +--------------+
            |  llm_call()  |  |  llm_call()  |  |  llm_call()  |
            | (Worker 1)   |  | (Worker 2)   |  | (Worker N)   |
            +--------------+  +--------------+  +--------------+
                    |                 |                 |
                    +--------+--------+--------+--------+
                             |
                             v
              +-------------------------------+
              |       synthesizer()           |
              | (Aggregation & Final Output)  |
              +-------------------------------+
                             |
                             v
                           END
```

**Flow Summary:**
1. START → orchestrator() → assign_workers() → [parallel llm_call() instances] → synthesizer() → END

---

## Function and Variable Reference

### Core Variables

**llm**
- Type: ChatGroq instance
- Purpose: The base language model used for all LLM operations throughout the workflow
- Initialized with model "llama-3.1-8b-instant"

**planner**
- Type: LLM with structured output
- Purpose: Enhanced version of llm that returns structured Sections objects instead of raw text
- Created by: `llm.with_structured_output(Sections)`

**orchestrator_worker_builder**
- Type: StateGraph
- Purpose: The graph builder that defines the workflow structure before compilation
- Used to: Add nodes and edges to construct the workflow

**orchestrator_worker**
- Type: Compiled graph
- Purpose: The final executable workflow that can be invoked
- Created by: `orchestrator_worker_builder.compile()`

### State Classes

**Section**
- Type: Pydantic BaseModel
- Fields: name (str), description (str)
- Purpose: Schema for a single report section with metadata

**Sections**
- Type: Pydantic BaseModel
- Fields: sections (List[Section])
- Purpose: Container schema for multiple Section objects returned by planner

**State**
- Type: TypedDict (Main workflow state)
- Fields:
  - topic (str): The report topic
  - sections (List[Section]): All planned sections from orchestrator
  - completed_sections (Annotated[list, operator.add]): Accumulator for finished sections
  - final_report (str): The synthesized output
- Purpose: Tracks the complete workflow state accessible to orchestrator and synthesizer

**WorkerState**
- Type: TypedDict (Worker-specific state)
- Fields:
  - section (Section): The single section this worker handles
  - completed_sections (Annotated[list, operator.add]): Access to shared accumulator
- Purpose: Provides minimal state needed for each worker to operate independently

### Node Functions

**orchestrator(state: State)**
- Input: State with topic field
- Processing:
  - Calls `planner.invoke()` with SystemMessage and HumanMessage
  - System prompt: "Generate a plan for the report."
  - Human message: Includes the topic
- Output: Returns dictionary with sections field containing List[Section]
- Role: Analyzes topic and generates dynamic plan with appropriate sections

**llm_call(state: WorkerState)**
- Input: WorkerState with single section assignment
- Processing:
  - Calls `llm.invoke()` with SystemMessage and HumanMessage
  - System prompt: Instructions for writing section with markdown formatting
  - Human message: Includes section name and description
- Output: Returns dictionary with completed_sections containing generated content
- Role: Writes content for one section and adds it to shared accumulator

**synthesizer(state: State)**
- Input: State with completed_sections accumulator
- Processing:
  - Retrieves all sections from completed_sections
  - Joins them with markdown separator `"\n\n---\n\n"`
- Output: Returns dictionary with final_report field containing complete document
- Role: Combines all worker outputs into cohesive final report

### Edge Functions

**assign_workers(state: State)**
- Input: State with sections list from orchestrator
- Processing:
  - Iterates through state["sections"]
  - Creates Send("llm_call", {"section": s}) for each section
- Output: Returns list of Send commands
- Role: Dynamically spawns parallel worker instances based on number of sections

### Graph Building Components

**orchestrator_worker_builder.add_node(name, function)**
- Adds nodes: "orchestrator", "llm_call", "synthesizer"
- Purpose: Registers functions as executable nodes in the graph

**orchestrator_worker_builder.add_edge(source, destination)**
- Edges: START → "orchestrator", "llm_call" → "synthesizer", "synthesizer" → END
- Purpose: Defines sequential connections between nodes

**orchestrator_worker_builder.add_conditional_edges(source, function, destinations)**
- Conditional edge: "orchestrator" → assign_workers → ["llm_call"]
- Purpose: Creates dynamic branching where assign_workers determines how many llm_call instances to spawn

---

## Workflow Components and Execution Flow

### 1. **Initialization Phase**

**Functions/Variables Involved:** `llm`

The workflow begins by setting up the foundational infrastructure:

- Environment variables are loaded to access API credentials
- The `llm` variable is created as a ChatGroq instance with model "llama-3.1-8b-instant"
- This `llm` will be used throughout the workflow for both orchestration and worker tasks

### 2. **Schema Definition Layer**

**Classes Involved:** `Section`, `Sections`
**Variables Involved:** `planner`

Two Pydantic model classes are defined to enforce structured output:

**Section class**: Defines the structure for a single report section with a name and description field. This ensures each section has consistent metadata.

**Sections class**: Acts as a container holding a list of Section objects. This top-level schema is what the orchestrator will use to return multiple sections in a structured format.

The `planner` variable is created by augmenting the base `llm` with structured output capability using `llm.with_structured_output(Sections)`. This forces the planner to return data matching the Sections schema rather than free-form text.

### 3. **State Management Architecture**

**Classes Involved:** `State`, `WorkerState`

Two distinct state schemas are defined to manage data flow:

**State class**: The complete state for the entire workflow containing:
- `topic` (str): The original topic for the report
- `sections` (List[Section]): The list of all planned sections generated by the orchestrator
- `completed_sections` (Annotated[list, operator.add]): A shared accumulator for completed sections using append operation
- `final_report` (str): The final synthesized report output

**WorkerState class**: A minimal subset containing only:
- `section` (Section): The specific section assigned to this worker
- `completed_sections` (Annotated[list, operator.add]): Access to the shared completed sections accumulator

This separation ensures workers operate independently while sharing results through a common accumulator.

### 4. **Orchestrator Node**

**Function:** `orchestrator(state: State)`
**Uses:** `planner` variable

The orchestrator function is the first node executed and serves as the planning brain:

**Input**: Receives State parameter with the report topic in `state['topic']`

**Processing**: 
- Calls `planner.invoke()` with a list containing SystemMessage and HumanMessage
- System prompt: "Generate a plan for the report."
- Human message: f"Here is the report topic: {state['topic']}"
- The planner returns a Sections object with structured section data

**Output**: Returns dictionary with key "sections" containing `report_sections.sections` (List[Section]). Each Section has a name and description representing subtasks for workers.

**Purpose**: Dynamically determines how to break down the complex report-writing task based on the specific topic provided.

### 5. **Worker Assignment Function**

**Function:** `assign_workers(state: State)`
**Uses:** `Send` API from LangGraph

After the orchestrator generates the plan, this conditional edge function dynamically creates worker tasks:

**Input**: Receives State parameter with all planned sections in `state["sections"]`

**Processing**: 
- Iterates through each section `s` in `state["sections"]`
- For each section, creates `Send("llm_call", {"section": s})` that will spawn a worker node
- Each Send command packages a single section into the WorkerState format

**Output**: Returns list comprehension `[Send("llm_call", {"section": s}) for s in state["sections"]]`

**Purpose**: Enables dynamic parallelization—the number of workers matches the number of sections the orchestrator decided to create, which varies based on the topic.

### 6. **Worker Nodes**

**Function:** `llm_call(state: WorkerState)`
**Uses:** `llm` variable

Multiple llm_call worker instances execute in parallel, each handling one section:

**Input**: Each worker receives WorkerState parameter with one assigned section in `state['section']`

**Processing**:
- Calls `llm.invoke()` with SystemMessage and HumanMessage
- System prompt: "Write a report section following the provided name and description. Include no preamble for each section. Use markdown formatting."
- Human message: f"Here is the section name: {state['section'].name} and description: {state['section'].description}"
- The llm generates the section content

**Output**: 
- Returns dictionary with key "completed_sections" containing `[section.content]`
- This writes the completed section to the shared completed_sections accumulator

**Purpose**: Parallelizes the actual content generation work. Each worker focuses on writing one section independently, allowing for faster execution than sequential processing.

### 7. **Synthesizer Node**

**Function:** `synthesizer(state: State)`

After all workers complete, the synthesizer creates the final output:

**Input**: Receives State parameter with all completed sections in `state["completed_sections"]`

**Processing**:
- Retrieves completed sections: `completed_sections = state["completed_sections"]`
- Joins all sections with markdown separators: `"\n\n---\n\n".join(completed_sections)`
- Stores result in `completed_report_sections` variable

**Output**: Returns dictionary with key "final_report" containing the `completed_report_sections` string

**Purpose**: Combines all parallel work into a cohesive final deliverable. This node could also perform additional synthesis like adding an executive summary or ensuring consistency across sections.

### 8. **Graph Construction**

**Variables:** `orchestrator_worker_builder`, `orchestrator_worker`
**Functions Used:** `orchestrator`, `llm_call`, `synthesizer`, `assign_workers`

The workflow is assembled using LangGraph's StateGraph:

**Graph Initialization**:
- `orchestrator_worker_builder = StateGraph(State)` creates the graph builder

**Nodes Added via add_node()**:
- `orchestrator_worker_builder.add_node("orchestrator", orchestrator)` - Planning node
- `orchestrator_worker_builder.add_node("llm_call", llm_call)` - Worker template that will be instantiated multiple times
- `orchestrator_worker_builder.add_node("synthesizer", synthesizer)` - Final assembly node

**Edge Configuration via add_edge() and add_conditional_edges()**:
- `add_edge(START, "orchestrator")` - Workflow begins with planning
- `add_conditional_edges("orchestrator", assign_workers, ["llm_call"])` - Dynamic worker assignment based on plan
- `add_edge("llm_call", "synthesizer")` - All workers write to synthesizer
- `add_edge("synthesizer", END)` - Workflow completes after synthesis

**Compilation**: 
- `orchestrator_worker = orchestrator_worker_builder.compile()` creates the final executable workflow that manages state transitions and parallel execution.

### 9. **Invocation and Execution Flow**

**Invocation:** `state = orchestrator_worker.invoke({"topic": "Create a report on Agentic AI RAGs"})`

When the workflow is invoked with a topic dictionary:

**Step 1 - Planning via orchestrator()**: 
- Initial state dictionary `{"topic": "Create a report on Agentic AI RAGs"}` is passed to orchestrator
- The orchestrator function calls `planner.invoke()` to analyze the topic
- Returns structured plan with multiple Section objects stored in `state["sections"]`

**Step 2 - Worker Assignment via assign_workers()**:
- The assign_workers function examines `state["sections"]`
- Creates one `Send("llm_call", {"section": s})` command per section
- LangGraph spawns multiple parallel llm_call worker instances

**Step 3 - Parallel Execution via llm_call()**:
- Each llm_call worker receives its assigned section in WorkerState
- Workers execute simultaneously, each calling `llm.invoke()` to write their section
- Each worker writes to `state["completed_sections"]` accumulator using operator.add

**Step 4 - Synthesis via synthesizer()**:
- Once all workers finish, synthesizer function is triggered
- Retrieves all sections from `state["completed_sections"]`
- Joins them with `"\n\n---\n\n".join()` into `state["final_report"]`

**Step 5 - Output**:
- The final report is accessed via `state["final_report"]`
- Can be rendered as formatted markdown using `Markdown(state["final_report"])`

---

## Key Design Principles

### Dynamic Parallelization
Unlike static parallel workflows where branches are pre-defined, this pattern determines the number and nature of parallel tasks at runtime based on the orchestrator's analysis.

### State Isolation with Shared Accumulation
Workers operate on isolated state subsets but write to a shared accumulator, preventing conflicts while enabling result aggregation.

### Structured Output Enforcement
Using Pydantic schemas ensures the orchestrator returns parsable, predictable data structures rather than free-form text that would require complex parsing.

### Separation of Concerns
- Orchestrator handles planning and task decomposition
- Workers handle execution of individual subtasks
- Synthesizer handles result aggregation and final formatting

---

## Workflow Characteristics

**Flexibility**: The orchestrator adapts to different topics by generating appropriate sections dynamically.

**Scalability**: Adding more sections doesn't require code changes—workers are created on-demand.

**Efficiency**: Parallel worker execution reduces total processing time compared to sequential section writing.

**Maintainability**: Clear separation between planning, execution, and synthesis makes each component easy to modify independently.
