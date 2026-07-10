# Voyager — Python Logic Flow Complete

Audited commit: `55e45a880755d0c8c66ca7fb5fe7962ac8974f89`

---

## 1. Entry surfaces

| API | Purpose |
|---|---|
| `Voyager.learn(reset_env=True)` | Lifelong open-ended learning |
| `Voyager.inference(task=..., sub_goals=..., reset_mode=...)` | Goal-conditioned run with decomposition |
| `Voyager.decompose_task(task)` | Curriculum decomposition only |
| `Voyager.reset` / `step` / `rollout` | Low-level task episode control |

---

## 2. `Voyager.learn` (outer lifelong loop)

```
if resume: env.reset(soft) else: env.reset(hard); resume=True
last_events = env.step("")
while True:
  if recorder.iteration > max_iterations: break
  task, context = curriculum_agent.propose_next_task(events, chest_obs, max_retries=5)
  try:
    messages, reward, done, info = rollout(task, context)
  except Exception:
    sleep(3); info={task, success:False}
    last_events = env.reset(hard, restore inventory/equipment/position)
  if info.success: skill_manager.add_new_skill(info)
  curriculum_agent.update_exploration_progress(info)
return {completed_tasks, failed_tasks, skills}
```

### Decisions
- resume soft vs hard inventory  
- iteration limit  
- success → add skill  
- exception → hard reset preserving inventory snapshot  

---

## 3. `rollout` / `reset` / `step` (iterative prompting)

### reset(task, context)
```
action_agent_rollout_num_iter = 0
optional env soft reset
difficulty = easy if completed_tasks > 15 else peaceful
events = env.step(time set + difficulty)
skills = skill_manager.retrieve_skills(query=context)
system = action.render_system_message(skills)
human = action.render_human_message(events, code="", task, context, critique="")
messages = [system, human]
```

### step()
```
ai = action.llm(messages)
parsed = action.process_ai_message(ai)
if parse OK (dict):
  code = program_code + exec_code
  events = env.step(code, programs=skill_manager.programs)
  recorder.record(events, task)
  action.update_chest_memory(nearbyChests)
  success, critique = critic.check_task_success(...)
  if reset_placed_if_failed and not success:
    revert placed blocks via givePlacedItemBack
  new_skills = retrieve_skills(context + chatlog_summary)
  rebuild system+human with code + critique
else:
  record empty; print parse error; keep messages
iter += 1
done = (iter >= task_max_retries) OR success
return messages, 0, done, info{task, success, conversations, program_* if success}
```

### Decisions
- parse fail vs execute  
- critic success vs fail  
- reset placed blocks?  
- done by success vs retry budget (`action_agent_task_max_retries`, default 4)  

---

## 4. ActionAgent

| Method | Logic |
|---|---|
| `render_system_message` | load action_template; inject primitives+skills; response format |
| `render_human_message` | fold events into observation fields |
| `process_ai_message` | extract JS fences; Babel parse; last async function(bot); build program_code + `await name(bot);` |
| `update_chest_memory` | merge/remove Invalid chests; dump JSON |
| `summarize_chatlog` | regex craft/mine missing requirements |

Parse retry: 3 attempts with sleep on Babel/assert failure.

---

## 5. CriticAgent

```
human = render_human_message(...)  # None if onError
if mode manual: human y/n + critique
if mode auto: llm → fix_and_parse_json → success, critique
  on parse fail: recurse max_retries-1; floor → False, ""
```

---

## 6. CurriculumAgent

### propose_next_task
1. If `progress==0` and auto → hard-coded `"Mine 1 wood log"`.  
2. If `inventoryUsed >= 33` → deposit to known chest / place chest / craft chest.  
3. Else LLM with curriculum system + warm-up-gated human; parse `Task:` line; `get_task_context`.  
4. Manual mode: human inputs.

### Warm-up
Observation key included only if `progress >= warm_up[key]`.  
For non-zero thresholds, 80% random include when unlocked.  
`optional_inventory_items`: filter inventory by core regex until threshold.

### QA
- Seed 3 biome questions + LLM questions (regex Question/Concept).  
- For each question: Chroma similarity < 0.05 → use cache; else answer LLM and store.  
- Context for curriculum uses ≤5 non-Unknown answers.

### update_exploration_progress
- Skip recording deposit tasks.  
- success → completed_tasks; else failed_tasks.  
- `clean_up_tasks`: dedupe completed; remove completed from failed; dump JSON.

### decompose_task
System decomposition prompt + human observation + `Final task:`; parse JSON list.

---

## 7. SkillManager

```
add_new_skill(info):
  skip deposit tasks
  description = LLM(skill prompt + code)
  if name exists: delete vectordb id; dump as nameV{i}
  vectordb.add_texts(description, id=name)
  skills[name] = {code, description}
  write code/description files; skills.json; persist

retrieve_skills(query):
  k = min(count, top_k)
  similarity_search → return codes
```

`programs` = all skill codes + control_primitives for env.

---

## 8. Inference loop

```
require task or sub_goals
sub_goals = decompose_task(task) if needed
env.reset(reset_mode)
clear completed/failed
while progress < len(sub_goals):
  next_task = sub_goals[progress]
  context = get_task_context(next_task)
  rollout(...)
  update_exploration_progress
```

Note: `progress` property = `len(completed_tasks)`; failed tasks still increment failed list but progress only grows on success — **stuck risk** if a subgoal repeatedly fails (by design of property).

---

## 9. Decision catalog

| ID | Decision | Location |
|---|---|---|
| D1 | resume soft vs hard reset | learn |
| D2 | iteration > max_iterations | learn |
| D3 | curriculum auto vs manual | CurriculumAgent |
| D4 | progress==0 hard-coded first task | propose_next_task |
| D5 | inventoryUsed≥33 force chest tasks | propose_next_task |
| D6 | warm-up gate + 80% sample | render_human_message |
| D7 | QA cache hit (score<0.05) | run_qa |
| D8 | critic auto vs manual | CriticAgent |
| D9 | action parse success | process_ai_message |
| D10 | critic success | check_task_success |
| D11 | reset_placed_if_failed | step |
| D12 | done = success or max retries | step |
| D13 | add skill on success | learn |
| D14 | skip deposit skill | add_new_skill |
| D15 | skill name collision versioning | add_new_skill |
| D16 | difficulty peaceful vs easy | reset |
| D17 | gpt-3.5 omits useChest/mineflayer context | render_system_message |
| D18 | inference needs task or sub_goals | inference |

---

## 10. Loop inventory

| Loop | Bound |
|---|---|
| learn while | max_iterations (recorder) |
| rollout while | success or task_max_retries |
| critic parse retry | max_retries (5) |
| curriculum propose retry | max_retries (5) |
| action Babel parse retry | 3 |
| QA over questions | questions list |
| inference over sub_goals | len(sub_goals) via progress |
| chest memory for-loops | chests dict |

---

## 11. I/O summary

| Stage | Input | Output |
|---|---|---|
| propose_next_task | events, chests | task, context |
| action step | messages | code / parse error |
| env.step | code, programs | events |
| critic | events, task, context | success, critique |
| add_new_skill | info program | skills.json + files + vectordb |
| retrieve_skills | query | list[code] |
| learn | — | completed/failed/skills |
| decompose | task, events | list[subgoal] |
