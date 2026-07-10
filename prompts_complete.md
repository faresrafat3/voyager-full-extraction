# Voyager — Complete Prompt Extraction

Source: `voyager/prompts/*.txt` + runtime human/system assembly in agents  
Commit: `55e45a880755d0c8c66ca7fb5fe7962ac8974f89`

Loader: `load_prompt(name)` → `voyager/prompts/{name}.txt`

---

## 1. Action agent

### 1.1 `action_template.txt` (system)

Role: write Mineflayer JS to complete Minecraft tasks.

Placeholders:
- `{programs}` — control primitive context + retrieved skill codes  
- `{response_format}` — from `action_response_format.txt`

Tells the model each round it will receive: last code, execution error, chat log, biome, time, nearby blocks/entities, health, hunger, position, equipment, inventory, chests, task, context, critique.

Requires response sections: **Explain** / **Plan** / **Code**, with strict coding rules (reuse primitives, no dig/craft directly, exploreUntil, maxDistance 32, no infinite loops, no bot.on, async `function(bot)`, …).

### 1.2 `action_response_format.txt`

```text
Explain: ...
Plan:
1) ...
2) ...
Code:
```javascript
// helpers if needed
async function yourMainFunctionName(bot) {
  // ...
}
```
```

### 1.3 Action human message (runtime, `ActionAgent.render_human_message`)

Assembled fields (not a static file):

| Field | Condition |
|---|---|
| Code from the last round | empty → "No code in the first round" |
| Execution error | if `execution_error` flag |
| Chat log | if `chat_log` flag |
| Biome, Time, Nearby blocks, Nearby entities | always from observe |
| Health, Hunger, Position, Equipment, Inventory | always |
| Chests | from chest_memory unless task is deposit-to-chest |
| Task, Context, Critique | always (Context/Critique may be "None") |

### 1.4 Control primitives injected into `{programs}`

From `control_primitives_context/`:  
`exploreUntil`, `mineBlock`, `craftItem`, `placeItem`, `smeltItem`, `killMob` (+ `useChest`, `mineflayer` if model ≠ gpt-3.5-turbo).

Plus top-k retrieved skill codes from SkillManager.

---

## 2. Critic agent — `critic.txt` (system)

Assesses whether task requirements are met. Exceeding requirements counts as success.

**Response JSON only:**
```json
{
  "reasoning": "reasoning",
  "success": true,
  "critique": "critique"
}
```

Few-shot examples cover: multi-log inventory, craft without crafting, raw_iron for iron_ore mining, planting via nearby blocks, kill zombie via rotten_flesh, eating via hunger=20, chest deposit inventory slots.

### Critic human message (runtime)

Biome, Time, Nearby blocks, Health, Hunger, Position, Equipment, Inventory, chest_observation, Task, Context.  
If any `onError` event → returns None → treated as failure.

---

## 3. Curriculum agent

### 3.1 `curriculum.txt` (system) — next task

Ultimate goal: discover diverse things, many tasks, best Minecraft player.

**Criteria (summary):** mentor-style; specific resource/craft/kill; single concise phrase; not too hard; novel; avoid shelter/build/place/plant/trade (hard to verify); avoid visual-only tasks.

**Format:**
```text
Reasoning: ...
Task: The next task.
```

Example: `Task: Obtain a wood log.`

### 3.2 `curriculum_qa_step1_ask_questions.txt`

Ask 5–10 Minecraft-specific self-contained questions with concepts. No building. No questions needing private inventory/nearby context without naming biome/item.

**Format:**
```text
Reasoning: ...
Question 1: ...
Concept 1: ...
...
```

### 3.3 `curriculum_qa_step2_answer_questions.txt`

```text
Answer: ...
```
or `Answer: Unknown`.

### 3.4 `curriculum_task_decomposition.txt`

Decompose final task + inventory into ordered subgoals.

**JSON list only:**
```json
["subgoal1", "subgoal2", ...]
```

Formats: Mine/Craft/Smelt/Kill/Cook/Equip with quantities; include tool tiers.

### 3.5 Curriculum human / context (runtime)

- Observation keys gated by **warm-up** thresholds (see logic doc).  
- Context block: up to 5 Q&A pairs (skip Unknown / "language model").  
- Hard-coded tasks: first task; inventory ≥33 deposit/chest craft.  
- `get_task_context(task)`: question `How to {task} in Minecraft?` + cached answer.

---

## 4. Skill manager — `skill.txt` (system)

Write a ≤6 sentence single-line description of the main Mineflayer function. Do not mention function name or bot.chat/helpers.

Runtime human: `program_code` + `The main function is `{name}`.`

Stored form:
```javascript
async function {name}(bot) {
    // {llm description}
}
```

---

## 5. Prompt wiring map

| Prompt file | Agent | Phase |
|---|---|---|
| action_template + action_response_format | ActionAgent | system each step |
| (human assembled) | ActionAgent | human each step |
| critic | CriticAgent | system check_task_success |
| (human assembled) | CriticAgent | human check |
| curriculum | CurriculumAgent | system propose_next_task |
| (human assembled + warm-up) | CurriculumAgent | human propose |
| curriculum_qa_step1_ask_questions | CurriculumAgent | QA step1 |
| curriculum_qa_step2_answer_questions | CurriculumAgent | QA step2 + get_task_context |
| curriculum_task_decomposition | CurriculumAgent | decompose_task / inference |
| skill | SkillManager | generate_skill_description |

---

## 6. Non-file but prompt-adjacent strings

- First task: `"Mine 1 wood log"` + wood-type context.  
- Deposit task templates with inventory slot requirements.  
- Parse curriculum: lines starting `Task:`.  
- Action parse: markdown JS fences; Babel AST; last async function named with param `bot`.  
- Chatlog summary filters: cannot make / need crafting table / need tool to mine → "I also need …".  
