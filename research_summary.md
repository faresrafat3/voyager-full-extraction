# Voyager — Research / System Summary

Paper: https://arxiv.org/abs/2305.16291  
Code: https://github.com/MineDojo/Voyager  
Audited commit: `55e45a880755d0c8c66ca7fb5fe7962ac8974f89`

## Core idea

Voyager is the first **LLM-powered embodied lifelong learning agent** in Minecraft that continuously:

1. explores via an **automatic curriculum**,  
2. accumulates an **ever-growing skill library** of executable code,  
3. improves programs through an **iterative prompting mechanism** that incorporates environment feedback, execution errors, and self-verification.

It queries GPT-4 (and GPT-3.5 for QA / skill description) as a black box — no weight fine-tuning.

## Three paper components ↔ code

| Paper | Code | Role |
|---|---|---|
| Automatic curriculum | `CurriculumAgent` | Propose next task + context from state, completed/failed history, optional QA |
| Skill library | `SkillManager` | Embed skill descriptions; retrieve top-k; add successful programs |
| Iterative prompting | `ActionAgent` + `CriticAgent` + env | Generate JS → execute → critique → retry up to `task_max_retries` |

## Main loops

### Lifelong learning — `Voyager.learn`
```
env reset (hard first / soft resume)
while recorder.iteration <= max_iterations:
  task, context = curriculum.propose_next_task(events, chests)
  messages, _, done, info = rollout(task, context)  # inner iterative prompting
  if success: skill_manager.add_new_skill(info)
  curriculum.update_exploration_progress(info)
```

### Single-task rollout — `Voyager.rollout` / `step`
```
reset(task, context): build system+human messages with retrieved skills
while not done:
  ai = action_agent.llm(messages)
  parse JS → env.step(code + programs)
  success, critique = critic.check_task_success(...)
  if reset_placed_if_failed and not success: revert placed blocks
  re-retrieve skills; rebuild messages with code/errors/critique
  done = success OR retries exhausted
```

### Inference — `Voyager.inference`
```
sub_goals = decompose_task(task)  # or provided
for next_task in sub_goals:
  context = get_task_context(next_task)
  rollout(next_task, context)
  update progress
```

## Agents in detail

### ActionAgent (iterative prompting)
- **System:** `action_template.txt` filled with control-primitive context + retrieved skills + response format.  
- **Human:** last code, execution errors, chat log, biome/time/blocks/entities/health/hunger/position/equipment/inventory/chests, task, context, critique.  
- **Parse:** extract ```javascript``` blocks via Babel; require last async `function(bot)`.  
- **Chest memory:** persists known chest inventories across steps.

### CriticAgent (self-verification)
- **System:** `critic.txt` → JSON `{reasoning, success, critique}`.  
- **Modes:** `auto` (LLM) or `manual` (human y/n).  
- Retries parse up to `max_retries` (default 5).  
- Returns success=False on env error events.

### CurriculumAgent (automatic curriculum)
- **System:** `curriculum.txt` → `Reasoning:` + `Task:` single phrase.  
- **Warm-up gates:** observations appear only after N completed tasks (default map in `default_warmup`).  
- **Hard-coded first task:** progress==0 → `"Mine 1 wood log"`.  
- **Inventory full (≥33):** force deposit/place/craft chest tasks.  
- **QA path:** step1 ask 5–10 questions (`curriculum_qa_step1`) + biome seed questions; step2 answer (`curriculum_qa_step2`); cache in Chroma + JSON.  
- **Context for a task:** `"How to {task} in Minecraft?"` via QA cache/LLM.  
- **Decompose:** `curriculum_task_decomposition.txt` → JSON list of subgoals.  
- **Modes:** auto / manual.

### SkillManager
- On success: generate description with `skill.txt`; store code+description; embed description; version file if name collision (`V2`, …).  
- Retrieve: top-k similar descriptions for query (context ± chat summary).  
- `programs` property = all skill codes + control primitives for env execution.  
- Skip adding deposit-chest skills.

## Environment

- Python `VoyagerEnv` bridges to Node Mineflayer server.  
- `env.step(code, programs=...)` returns event list ending with `observe`.  
- Events: `onChat`, `onError`, `onSave`, `observe` (status, inventory, voxels, chests, …).  
- Difficulty: peaceful until >15 completed tasks, then easy.  
- Soft vs hard reset (keep vs clear inventory).

## Control primitives (context shown to Action LLM)

Always: `exploreUntil`, `mineBlock`, `craftItem`, `placeItem`, `smeltItem`, `killMob`.  
If not gpt-3.5-turbo: also `useChest`, `mineflayer`.

## Headline paper claims (context)

- 3.3× more unique items, 2.3× longer distance, up to 15.3× faster tech-tree milestones vs prior SOTA.  
- Skills transfer to new worlds for novel tasks.

## Strengths for extraction / ARSENAL

- Clean multi-agent prompt surface (8 prompt files).  
- Explicit lifelong loop + skill compounding (not covered by ToT/LATS/Reflexion alone).  
- Maps to ARSENAL: curriculum ≈ outer goal gen; action+critic ≈ refine loop; skill library ≈ persistent procedural memory.

## Inputs / Outputs

**Inputs:** Minecraft connection, OpenAI key, model names, max_iterations, ckpt/skill dirs, resume.  
**Outputs:** completed/failed task lists, skills dict (code+description), event recordings, checkpoints.
