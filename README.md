# Data Assistant AI Agent

An AI-powered **Data Assistant Agent** that accepts a user's data-related goal, creates a plan, converts the plan into executable actions, uses tools to work with CSV data, maintains state, and iteratively decides what to do next until the task is completed.

The project demonstrates the core concepts of an **AI Agent**:

* Reasoning
* Planning
* Tool selection
* Tool execution
* Observation
* State management
* Iterative decision-making
* Final response generation

---

## Workflow

The agent follows this workflow:

```text
                    USER GOAL
                        │
                        ▼
                  ┌───────────┐
                  │  PLANNER  │
                  └─────┬─────┘
                        │
                        ▼
                       PLAN
                        │
                        ▼
              ┌──────────────────┐
              │ PLAN → ACTIONS   │
              └────────┬─────────┘
                       │
                       ▼
                 ALL ACTIONS
                       │
                       ▼
                INITIALIZE STATE
                       │
                       ▼
              ┌───────────────────┐
              │    AGENT LOOP     │
              │                   │
              │      DECIDE       │
              │        ↓          │
              │      EXECUTE      │
              │        ↓          │
              │      OBSERVE      │
              │        ↓          │
              │   UPDATE STATE    │
              │        ↓          │
              │   DECIDE AGAIN    │
              │        │          │
              └────────┼──────────┘
                       │
                       ▼
                     FINISH
                       │
                       ▼
                 FINAL ANSWER
```

The important part is that the agent does **not** simply ask the LLM for an answer.

Instead, it repeatedly follows:

```text
DECIDE → EXECUTE → OBSERVE → UPDATE STATE → DECIDE AGAIN
```

---

## Example User Goal

For example, the user can provide:

```text
Find the top 3 products by total revenue from the sales CSV
and save the result as a CSV file.
```

The agent then determines what needs to be done instead of directly generating an answer.

---

## Agent Architecture

### 1. User Goal

The user provides a natural-language task.

```text
Find the top 3 products by total revenue
and save the result as a CSV file.
```

The goal is stored in the agent's state and passed to the planning component.

---

### 2. Planner

The planner uses the LLM to break the goal into logical steps.

Example:

```text
1. Load the CSV dataset.
2. Inspect the dataset.
3. Check whether revenue can be calculated.
4. Calculate revenue.
5. Aggregate revenue by product.
6. Find the top 3 products.
7. Save the result as a CSV file.
8. Verify the generated file.
```

The planner does **not** execute any tools.

Its responsibility is to answer:

> "What needs to happen to complete this task?"

---

### 3. Plan → Actions

The high-level plan is then converted into executable actions.

Example:

```json
[
    {
        "step": 1,
        "action": "read_csv",
        "purpose": "Load the dataset"
    },
    {
        "step": 2,
        "action": "inspect_dataset",
        "purpose": "Inspect columns and data"
    },
    {
        "step": 3,
        "action": "calculate_revenue",
        "purpose": "Calculate revenue from quantity and price"
    },
    {
        "step": 4,
        "action": "top_products",
        "purpose": "Find the top 3 products"
    },
    {
        "step": 5,
        "action": "save_csv",
        "purpose": "Save the result"
    },
    {
        "step": 6,
        "action": "verify_output",
        "purpose": "Verify the generated file"
    }
]
```

This stage does not execute the actions.

It converts:

```text
PLAN
```

into:

```text
EXECUTABLE ACTIONS
```

---

## 4. State Management

The agent maintains state throughout the workflow.

Example:

```python
state = {
    "goal": user_goal,
    "plan": plan,
    "actions": actions,
    "completed_actions": [],
    "observations": [],
    "results": {}
}
```

The state allows the agent to remember:

* What the user asked
* What plan was created
* What actions are available
* Which actions have already been executed
* What each tool returned
* Intermediate results
* Generated output files

Without state, the agent would not have reliable context about what happened during previous steps.

---

# Functions

The project is divided into several functions, each responsible for a specific part of the agent.

## `read_csv()`

Reads the input CSV file.

```python
def read_csv():
    df = pd.read_csv("sales.csv")
    return df
```

Purpose:

```text
CSV file
   ↓
Pandas DataFrame
```

The resulting dataset is stored in the agent's state.

---

## `inspect_dataset(df)`

Inspects the dataset before performing analysis.

```python
def inspect_dataset(df):
    return {
        "columns": list(df.columns),
        "row_count": len(df),
        "data_types": df.dtypes.astype(str).to_dict(),
        "missing_values": df.isnull().sum().to_dict(),
        "sample_rows": df.head(5).to_dict(orient="records")
    }
```

The agent can use this information to determine whether the dataset actually contains the information required by the user's goal.

For example:

```text
User asks:
"Which product has the highest revenue?"
```

If the dataset contains:

```text
product
quantity
price
```

then revenue can be calculated.

But if the user asks:

```text
"Which product has the highest profit margin?"
```

and the dataset does not contain cost/profit information, the agent should not invent an answer.

Instead, it should identify the missing information.

---

## `calculate_revenue(df)`

Calculates revenue from quantity and price.

```python
def calculate_revenue(df):
    df = df.copy()

    df["quantity"] = pd.to_numeric(
        df["quantity"],
        errors="coerce"
    )

    df["price"] = pd.to_numeric(
        df["price"],
        errors="coerce"
    )

    df["revenue"] = df["quantity"] * df["price"]

    return df
```

This produces a new `revenue` column.

Example:

```text
quantity × price = revenue
```

---

## `top_products(df)`

Aggregates revenue by product and returns the top three products.

```python
def top_products(df):
    result = (
        df.groupby("product")["revenue"]
        .sum()
        .reset_index()
        .rename(
            columns={"revenue": "total_revenue"}
        )
        .sort_values(
            "total_revenue",
            ascending=False
        )
        .head(3)
    )

    return result
```

Example output:

```text
product    total_revenue
Phone      240000
Laptop     180000
Tablet     100000
```

---

## `save_csv(df)`

Saves the generated result.

```python
def save_csv(df):
    output_path = "top_3_products_by_revenue.csv"

    df.to_csv(
        output_path,
        index=False
    )

    return {
        "status": "success",
        "file": output_path,
        "rows_saved": len(df)
    }
```

This allows the agent to perform an actual action rather than only returning text.

---

## `verify_output(file_path)`

Verifies the generated output file.

```python
def verify_output(file_path):
    df = pd.read_csv(file_path)

    return {
        "file_exists": True,
        "rows": len(df),
        "columns": list(df.columns),
        "data": df.to_dict(
            orient="records"
        )
    }
```

This provides a final verification step before the agent finishes.

---

# Plan → Actions Function

The `convert_plan_to_actions()` function converts the LLM-generated plan into actions that the system can actually execute.

```python
def convert_plan_to_actions(plan):
    ...
```

The function provides the LLM with the list of available tools.

The LLM is instructed to:

* Use only available tools
* Never invent tools
* Not execute tools
* Convert the entire plan into executable actions
* Preserve the logical order

Example:

```text
PLAN

    ↓

convert_plan_to_actions()

    ↓

read_csv
inspect_dataset
calculate_revenue
top_products
save_csv
verify_output
```

---

# Decision Maker

The `decide_next_action()` function is responsible for selecting **one action at a time**.

```python
def decide_next_action(state):
    ...
```

The decision maker receives the current state and determines:

> "What should the agent do right now?"

For example:

```text
State:
No actions completed.

Decision:
read_csv
```

After execution:

```text
State:
read_csv completed.

Decision:
inspect_dataset
```

Then:

```text
State:
read_csv
inspect_dataset completed.

Decision:
calculate_revenue
```

This continues until the task is complete.

---

# Executor

The `execute_action()` function actually runs the selected tool.

```python
def execute_action(action, state):
    ...
```

It connects the LLM's decision to real Python functions.

For example:

```text
LLM decides:

"calculate_revenue"

        ↓

execute_action()

        ↓

calculate_revenue(df)

        ↓

Result stored in state
```

The LLM decides **what to do**.

Python performs **the actual operation**.

This separation is important for building a reliable agent.

---

# Agent Loop

The agent loop connects all components together.

```python
while True:

    decision = decide_next_action(state)

    action = decision["action"]

    if action == "finish":
        break

    result = execute_action(
        action,
        state
    )

    state["completed_actions"].append(
        action
    )

    state["observations"].append({
        "action": action,
        "result": str(result)
    })
```

The loop follows:

```text
              ┌──────────────┐
              │    DECIDE    │
              └──────┬───────┘
                     ↓
              ┌──────────────┐
              │   EXECUTE    │
              └──────┬───────┘
                     ↓
              ┌──────────────┐
              │   OBSERVE    │
              └──────┬───────┘
                     ↓
              ┌──────────────┐
              │ UPDATE STATE │
              └──────┬───────┘
                     ↓
                  DECIDE
                     │
                     └──────→ ...
```

The loop stops when the decision maker returns:

```json
{
    "action": "finish",
    "reason": "All required actions have been completed."
}
```

---

# Tool Registry

The available tools can be maintained in a registry:

```python
tool_registry = {
    "read_csv": read_csv,
    "inspect_dataset": inspect_dataset,
    "calculate_revenue": calculate_revenue,
    "top_products": top_products,
    "save_csv": save_csv,
    "verify_output": verify_output
}
```

This gives the agent a controlled set of operations that it is allowed to perform.

The LLM cannot directly execute arbitrary Python code. It selects from the tools exposed by the application.

---

# Complete Agent Flow

For the example task, the actual execution becomes:

```text
User:
"Find the top 3 products by total revenue
and save the result as a CSV."

                ↓

            PLANNER

                ↓

              PLAN

                ↓

        PLAN → ACTIONS

                ↓

        Initialize State

                ↓

        ┌─────────────────┐
        │  DECIDE         │
        │  read_csv       │
        └────────┬────────┘
                 ↓
             EXECUTE
                 ↓
          CSV loaded
                 ↓
          UPDATE STATE
                 ↓
              DECIDE
                 ↓
        inspect_dataset
                 ↓
             EXECUTE
                 ↓
         Dataset observed
                 ↓
          UPDATE STATE
                 ↓
              DECIDE
                 ↓
        calculate_revenue
                 ↓
             EXECUTE
                 ↓
        Revenue calculated
                 ↓
          UPDATE STATE
                 ↓
              DECIDE
                 ↓
          top_products
                 ↓
             EXECUTE
                 ↓
        Top 3 calculated
                 ↓
          UPDATE STATE
                 ↓
              DECIDE
                 ↓
            save_csv
                 ↓
             EXECUTE
                 ↓
          Output saved
                 ↓
          UPDATE STATE
                 ↓
              DECIDE
                 ↓
          verify_output
                 ↓
             EXECUTE
                 ↓
           Output verified
                 ↓
              DECIDE
                 ↓
              FINISH
                 ↓
           FINAL ANSWER
```

---

# Handling Missing Information

The agent should not blindly answer every question.

Before performing analysis, it should inspect the available data and determine whether the required information exists.

For example:

### Supported

```text
"Which product has the highest revenue?"
```

If the dataset contains:

```text
product
quantity
price
```

the agent can calculate:

```text
revenue = quantity × price
```

### Missing Information

```text
"Which product has the highest profit margin?"
```

If the dataset does not contain:

```text
cost
profit
```

the agent should not invent the information.

Instead, the workflow should become:

```text
USER GOAL
   ↓
INSPECT DATA
   ↓
REQUIRED INFORMATION MISSING
   ↓
REQUEST INFORMATION
   ↓
TELL USER WHAT IS MISSING
```

This makes the agent more reliable and prevents unsupported conclusions.

---

# Technologies Used

* Python
* Pandas
* Groq API
* LLM
* Google Colab
* CSV
* JSON

---

# Project Structure

```text
data-assistant-agent/
│
├── Data_Assistant_Agent.ipynb
├── sales.csv
├── top_3_products_by_revenue.csv
├── README.md
└── requirements.txt
```

---

# Running the Project

### 1. Open the notebook

Open:

```text
Data_Assistant_Agent.ipynb
```

in Google Colab.

### 2. Install dependencies

```python
!pip install -q groq pandas
```

### 3. Configure Groq

Add your Groq API key locally in Colab.

Do **not** commit the API key to GitHub.

### 4. Run the notebook

Execute the cells in order.

The notebook will:

```text
Load dataset
    ↓
Understand user goal
    ↓
Generate plan
    ↓
Convert plan into actions
    ↓
Initialize state
    ↓
Run agent loop
    ↓
Execute tools
    ↓
Update state
    ↓
Generate output
```

---

# Key Agentic Concepts Demonstrated

This project demonstrates the following concepts:

| Concept                      | Implementation                       |
| ---------------------------- | ------------------------------------ |
| User Goal                    | Natural-language task                |
| Planning                     | `planner`                            |
| Plan → Actions               | `convert_plan_to_actions()`          |
| Tool Selection               | `decide_next_action()`               |
| Tool Execution               | `execute_action()`                   |
| Observation                  | Tool results                         |
| State                        | `state` dictionary                   |
| Iterative Reasoning          | Agent loop                           |
| Data Analysis                | Pandas tools                         |
| Output Generation            | `save_csv()`                         |
| Verification                 | `verify_output()`                    |
| Missing Information Handling | Dataset inspection + decision making |

---

# Why This Is an AI Agent

This project is not simply:

```text
User → LLM → Answer
```

Instead, it follows an iterative workflow:

```text
User Goal
    ↓
Planning
    ↓
Action Selection
    ↓
Tool Execution
    ↓
Observation
    ↓
State Update
    ↓
Action Selection
    ↓
Tool Execution
    ↓
...
    ↓
Final Answer
```

The LLM is responsible for reasoning about the workflow and selecting actions, while deterministic Python functions perform the actual data operations.

---

# Future Improvements

Possible extensions include:

* Support for multiple CSV files
* More generic data-analysis tools
* Automatic chart generation
* Data cleaning tools
* Statistical analysis tools
* Natural-language report generation
* File upload support
* Search/web tools
* More robust error handling
* Human approval before sensitive actions
* LangGraph-based orchestration
* Persistent agent memory
* Support for Excel and other data formats

---

## Author

**Anisha Gaur**

Built as an exploration of **AI Agents, LLM reasoning, planning, tool use, and data analysis**. along with setup and running
