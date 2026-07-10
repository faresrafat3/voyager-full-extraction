# Voyager — Deep Dive Task / Phase Matrix

## Universal lifelong pipeline

| Phase | Code | Input | Output | Loop | Key decision | Stop |
|---|---|---|---|---|---|---|
| Init | `Voyager.__init__` | ports, keys, models, dirs | agents + env | no | resume skill/ckpt | — |
| Learn reset | `learn` | resume flag | env state | no | soft vs hard | — |
| Propose task | `CurriculumAgent.propose_next_task` | events, chests | task, context | retries | first task / full inv / auto LLM | max_retries |
| Retrieve skills | `SkillManager.retrieve_skills` | context query | code list | no | k=0 empty | — |
| Build messages | `ActionAgent.render_*` | skills, events | system+human | no | primitive set by model | — |
| Generate code | `ActionAgent.llm` + parse | messages | program_code, exec | parse retry 3 | valid async bot fn? | — |
| Execute | `env.step` | code, programs | events | no | timeout | — |
| Verify | `CriticAgent.check_task_success` | events, task | success, critique | parse retries | auto/manual; onError | max_retries |
| Revert place | `givePlacedItemBack` | placed blocks | updated inv/voxels | no | reset_placed_if_failed | — |
| Retry / done | `step` | success, iter | done flag | task_max_retries | success or budget | — |
| Add skill | `add_new_skill` | info | skills.json | no | skip deposit; rename Vn | — |
| Update curriculum | `update_exploration_progress` | info | completed/failed lists | no | skip deposit record | — |
| Outer stop | `learn` | recorder.iteration | break | max_iterations | — | limit |

---

## 1. Automatic curriculum

| Item | Detail |
|---|---|
| Prompt | `curriculum.txt` |
| First task | `Mine 1 wood log` if progress==0 |
| Full inventory | ≥33 slots → deposit/place/craft chest |
| Warm-up | unlock biome/time/QA context etc. by completed count |
| QA | step1 questions + step2 answers; Chroma cache score&lt;0.05 |
| Context | How to {task} in Minecraft? |
| Decomposition | JSON subgoal list for inference |
| Progress metric | `len(completed_tasks)` |

---

## 2. Iterative prompting (Action + Critic)

| Item | Detail |
|---|---|
| System | action_template + programs + response_format |
| Human | code, errors, chat, full state, task, context, critique |
| Parse | markdown JS → Babel → last async function(bot) |
| Critic JSON | reasoning, success, critique |
| Retries | default `action_agent_task_max_retries=4` |
| Feedback channels | execution error, chat log, critic critique, new skill retrieval |

---

## 3. Skill library

| Item | Detail |
|---|---|
| On success | describe with skill.txt; embed; store code |
| Retrieval | top_k (default 5) by description similarity to query |
| Query sources | task context; context + chatlog summary after step |
| Versioning | name collision → file `nameV{i}.js` but vectordb id = program_name |
| Execution bundle | `skill_manager.programs` = all skills + control primitives |

---

## 4. Inference mode

| Step | Detail |
|---|---|
| Require | task or sub_goals |
| Decompose | if only task |
| Loop | sub_goals[progress] with get_task_context |
| Reset | hard by default |
| Note | progress only advances on completed_tasks success |

---

## 5. Environment contract

| Mode | Meaning |
|---|---|
| soft reset | keep inventory |
| hard reset | clear inventory (or restore snapshot fields on error path) |
| observe event | authoritative state for agents |
| onChat / onError / onSave | side channels for action/critic |

---

## 6. Mapping to other extractions / ARSENAL

| Voyager piece | Closest prior pattern | ARSENAL note |
|---|---|---|
| Curriculum propose | Meta conductor goal split / L0 routing | open-ended goal generation |
| Action retry + critic | Self-Refine + Reflexion feedback | env-grounded refine |
| Skill library | — (new) | persistent procedural memory |
| learn outer loop | Reflexion trials / AI Scientist stages | lifelong continuum |
| vs LATS | MCTS episode search | Voyager compounds skills across tasks |

---

## 7. Default hyperparameter cheat sheet

```
max_iterations = 160
action_agent_task_max_retries = 4
action model = gpt-4, temp 0
curriculum model = gpt-4; qa = gpt-3.5-turbo
critic = gpt-4 auto
skill manager = gpt-3.5-turbo, retrieval_top_k = 5
env_wait_ticks = 20
env_request_timeout = 600
```
