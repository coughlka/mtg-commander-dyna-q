# CLAUDE.md

Guidance for Claude Code when working in this repo.

## What this project is

Penn State MSAI reinforcement learning course project. A tabular Dyna-Q agent that builds 100-card MTG Commander decks from a real, constrained collection. Deliverables are weekly, and the final submission is a single Jupyter notebook plus a course-template `.docx` writeup.

The full RL formulation lives in [week1_rl_formulation.md](week1_rl_formulation.md). Read it before making non-trivial changes to state, action, or reward design.

## Where the code lives

All implementation lives in `mtg_commander_dyna_q.ipynb`. There are no separate `.py` modules for agent, env, reward, etc. When the user asks to change module behavior, the change goes in a notebook cell.

`populate_template.py` is a one-off helper that fills the course `.docx` template with Week 1 content. It is not part of the RL pipeline.

## Cache directory

`cache/` is gitignored and roughly 500MB. It holds:

- `scryfall_bulk.pkl`, `collection_enriched.pkl` - card metadata
- `commander_profile_*.json` - per-commander profiles from the API
- `checkpoint.pkl`, `eval_results.pkl` - training outputs
- `metric*.png`, `api_validation.png` - generated plots

Never commit anything under `cache/`. If the user asks to regenerate a plot or rerun training, it goes in `cache/`.

## External API

Card data and deck scoring come from `https://mtg-assistant.up.railway.app`:

- `/synergy/commander-profile` - commander archetype profile (cached per commander)
- `/synergy/analyze-deck` - composite deck score, used as terminal reward

Training episodes must not call the API. The API is hit once during setup to build the cache, and training reads from cache only.

## Course deliverables

The notebook is the source of truth for code. Results get exported to rendered copies (`.html`, `.pdf`, `_executed.ipynb`) and summarized into the course `.docx` files (`Assignment_4_Results.docx`, `Assignment_4_5_Results.docx`, `RL Project Writeup.docx`). When updating results, regenerate the rendered notebook copies alongside the `.ipynb`.

## Writing style

- Never use em dashes or semicolons in prose, including in notebook markdown cells, commit messages, and docs. Use commas, periods, or parentheses instead.
- Keep comments minimal. The notebook is already heavily narrated in markdown cells, code comments should explain only non-obvious why.

## Scope discipline

This is a course project with a fixed RL formulation. Do not refactor the agent, env, or reward structure unless the user explicitly asks. Small bug fixes and plot tweaks are fine. Architectural changes need a conversation first.
