Backward Chaining Planner v2
A lightweight, zero-dependency AI planner written in Python. Combines Prolog-style unification with STRIPS/PDDL planning and HTN task decomposition in a single file.

Features
Variable Unification — Prolog-style logic variables (?x, ?y) are automatically bound through unification during planning
Backward Chaining Search — goal-directed DFS: starts from the goal and works backward to find which actions are needed
A* Cost-Optimal Search — finds the cheapest plan when actions have different costs
Delete Effects — actions can remove facts from the world state (e.g., moving a robot removes it from the old location)
HTN Decomposition — abstract tasks (like "deliver order") are broken down into subtask sequences through Methods
Diagnostic Feedback — when planning fails, returns exactly which goals are unsatisfied, which preconditions failed, and concrete suggestions for fixing the model
LLM Tool Interface — plan_tool() accepts JSON-friendly dicts and returns structured results, ready to be registered as a function tool for Claude, GPT, Gemini, or Grok
Quick Start
# Run all 7 built-in demos
python planner.py

# Interactive mode
python planner.py --inter

# Pipe JSON from stdin
echo '{"goals": [...], "actions": [...], "state": [...]}' | python planner.py --json
Requirements
Python 3.8+ (tested on 3.12)
No external dependencies
Core Concepts
Actions
An Action has a name, parameters (logic variables), preconditions, add effects, delete effects, and a cost:

Action("move", ["?from", "?to"],
       preconditions=[("at_robot", "?from"), ("connected", "?from", "?to")],
       effects=[("at_robot", "?to")],
       delete_effects=[("at_robot", "?from")],
       cost=1.0)
Methods (HTN)
A Method decomposes an abstract task into a sequence of subtasks:

Method("transport_obj",
       task=("transport", "?obj", "?from", "?to"),
       params=["?obj", "?from", "?to"],
       preconditions=[("at_robot", "?from"), ("at_obj", "?obj", "?from")],
       subtasks=[
           ("holding", "?obj"),
           ("at_robot", "?to"),
           ("at_obj", "?obj", "?to"),
       ])
State
A set of ground tuples representing facts about the world:

state = {("at_robot", "room_a"), ("hand_empty",), ("connected", "room_a", "room_b")}
API
backward_chain(goals, actions, state, depth=20)
DFS backward chaining. Returns a PlanResult.

backward_chain_astar(goals, actions, state)
A* search for the lowest-cost plan. Returns a PlanResult.

htn_plan(tasks, actions, methods, state, depth=30)
HTN planner that decomposes abstract tasks through methods. Returns a PlanResult.

plan_tool(goals, actions, state, mode, max_depth, methods)
JSON-friendly entry point for LLM tool use. Accepts lists/dicts, returns a dict.

PlanResult
.success — bool
.plan — list of (action_name, bindings) tuples
.cost — total cost (A* mode)
.diagnosis — Diagnosis object with failure details
.summary() — human-readable output
.to_dict() — structured dict for programmatic use
Built-in Demos
#	Scenario	Tests
1	Build pipeline (compile, link, test)	Ground terms, basic backward chaining
2	Download-compile-link-test	Variable unification across action chain
3	Pizza delivery (missing ingredient)	Failure diagnostics with suggestions
4	Fast vs optimized compilation	A* cost-optimal search
5	Robot moves box between rooms	Delete effects + HTN decomposition
6	Travel planning (taxi vs flight)	HTN with multiple methods
7	Restaurant order fulfillment	HTN full pipeline (5-step decomposition)
Using as an LLM Tool
Register plan_tool as a function tool. The LLM describes the planning problem in natural language, you (or the LLM) convert it to JSON, and the planner returns a guaranteed-valid plan or actionable diagnostics.

from planner import plan_tool

result = plan_tool(
    goal_descriptions=[["delivered", "?who", "?where"]],
    action_definitions=[
        {"name": "bake", "params": ["?c"],
         "preconditions": [["dough_ready"], ["oven_hot"]],
         "effects": [["pizza_ready", "?c"]],
         "delete_effects": [["dough_ready"]]},
        # ... more actions
    ],
    initial_state=[["oven_hot"], ["flour_available"]],
    mode="dfs"  # or "astar" or "htn"
)
# result is a dict: {"status": "success", "plan": [...]} or {"status": "failure", "suggestions": [...]}
Architecture
User / LLM
    |
    v
plan_tool()  <-- JSON in, JSON out
    |
    +---> backward_chain()      (DFS, goal-directed)
    +---> backward_chain_astar() (A*, cost-optimal)
    +---> htn_plan()            (HTN decomposition)
             |
             v
        Unification Engine (walk, unify, apply_subst)
             |
             v
        State Manager (apply_action_effects, delete effects)
             |
             v
        Diagnostics (diagnose_failure -> suggestions)
