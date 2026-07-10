# Voyager — Full Prompt, Logic, Flow, and Graph Extraction

Paper: [Voyager: An Open-Ended Embodied Agent with Large Language Models](https://arxiv.org/abs/2305.16291)  
Code: https://github.com/MineDojo/Voyager  
Audited commit: `55e45a880755d0c8c66ca7fb5fe7962ac8974f89`  
GitHub stars: ~7,037 (verified 2026-07-10) · License: MIT

## Why this package

Voyager is an **open-ended embodied lifelong learning agent** in Minecraft with three paper pillars:

1. **Automatic curriculum** — propose the next novel/doable task from state + QA context  
2. **Skill library** — store successful programs with natural-language descriptions; retrieve by embedding  
3. **Iterative prompting** — Action agent writes Mineflayer JS, executes in env, gets errors/chat/critique, retries  

This extraction maps every prompt, loop, decision, and I/O surface to the same standard as  
https://github.com/faresrafat3/lats-full-extraction

## Package contents

```text
voyager-full-extraction/
├── README.md
├── research_summary.md
├── prompts_complete.md
├── python_logic_flow_complete.md
├── python_logic_inventory.json
├── deep_dive_task_matrix.md
├── graph_english.md / .mmd
├── graph_arabic.md / .mmd
├── final_completeness_check_ar.md
├── QUALITY_REVIEW_AR.md
├── raw_prompt_files/          # prompts + agents + voyager.py + control primitives context
├── raw_data_samples/          # skill library samples
└── archives/voyager_full_extract.zip
```

## Core components (code map)

| Paper concept | Class / module |
|---|---|
| Main loop / lifelong learn | `voyager/voyager.py` → `Voyager.learn` |
| Iterative prompting (Action) | `agents/action.py` → `ActionAgent` |
| Self-verification (Critic) | `agents/critic.py` → `CriticAgent` |
| Automatic curriculum | `agents/curriculum.py` → `CurriculumAgent` |
| Skill library | `agents/skill.py` → `SkillManager` |
| Prompt templates | `prompts/*.txt` |
| Env bridge | `env/bridge.py` + Mineflayer Node |
| Control primitives | `control_primitives/` + `control_primitives_context/` |

## Modes

| Mode | Entry | Behavior |
|---|---|---|
| Lifelong learning | `voyager.learn()` | curriculum proposes tasks → rollout → skill add |
| Inference | `voyager.inference(task=...)` | decompose task → sequential sub-goal rollouts |
| Manual curriculum/critic | `mode="manual"` | human proposes tasks / judges success |

## Recommended reading order

1. `research_summary.md`  
2. `deep_dive_task_matrix.md`  
3. `prompts_complete.md`  
4. `python_logic_flow_complete.md`  
5. `graph_english.md` / `graph_arabic.md`  
6. `final_completeness_check_ar.md`  
7. `QUALITY_REVIEW_AR.md`  

## Related extractions

- LATS: https://github.com/faresrafat3/lats-full-extraction  
- ToT: https://github.com/faresrafat3/tot-full-extraction  
- Reflexion: https://github.com/faresrafat3/reflexion-full-extraction  
- ARSENAL: https://github.com/faresrafat3/arsenal-unified-master-pipeline  
- Consolidated: https://github.com/faresrafat3/llm-agent-research-extractions  

## Quality policy

- Source code is the source of truth.  
- Paper used as research context.  
- Skill library samples are illustrative; full trial libraries remain upstream.  
- Mermaid graphs are high-level maps; line-level detail is in Markdown/JSON inventories.  
