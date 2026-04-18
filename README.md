# Dyna-Q Reinforcement Learning for MTG Commander Deck Construction

Penn State MSAI reinforcement learning course project. A model-based Dyna-Q agent that learns to construct competitive Magic: The Gathering Commander decks from a real, constrained physical card collection.

**Author:** Keith Coughlin (krc5765@psu.edu)

## Research Question

Can a model-based RL agent learn to construct competitive Commander decks from a constrained physical collection using only card category abstractions and coarse state representations?

## Approach

- **Agent:** Tabular Dyna-Q with epsilon-greedy policy, Q-table, and learned transition model
- **Environment:** `gymnasium.Env` that fills 99 deck slots one at a time for a chosen commander
- **State:** 6 deck composition metrics (mana curve, removal density, card advantage density, synergy concentration, ramp density, slot progress), each discretized into 3 bins -> 729 states
- **Actions:** ~50 card categories (functional roles like `ramp_spell`, `spot_removal_creature`, `land_fetch`). A downstream heuristic selects the specific card from the collection.
- **Reward:** Intermediate shaping rewards during the episode plus a terminal reward from the `/synergy/analyze-deck` API endpoint
- **Constraints:** Singleton, color identity, and collection-bound, enforced via action masking

See [week1_rl_formulation.md](week1_rl_formulation.md) for the full RL formulation.

## Repository Layout

```
.
├── mtg_commander_dyna_q.ipynb    # Main notebook (all modules + training + eval)
├── mtg_commander_dyna_q.html     # Rendered notebook (HTML)
├── mtg_commander_dyna_q.pdf      # Rendered notebook (PDF)
├── week1_rl_formulation.md       # Week 1 deliverable, RL formulation writeup
├── populate_template.py          # Helper script to populate the course .docx template
├── Reinforcement_Learning_project template.docx
├── Reinforcement_Learning_Project_Week1.docx
├── Assignment_4_Results.docx
├── Assignment_4_5_Results.docx
├── RL Project Writeup.docx
├── CLAUDE.md                     # Instructions for Claude Code
└── cache/                        # (gitignored) API responses, enriched collection, checkpoints, plots
```

## Cache Directory

The `cache/` directory holds data that is expensive or slow to regenerate but not worth versioning:

- `scryfall_bulk.pkl`, `collection_enriched.pkl` - cached card metadata
- `commander_profile_*.json` - per-commander archetype profiles from the API
- `checkpoint.pkl`, `eval_results.pkl` - training checkpoints and eval output
- `metric*.png`, `api_validation.png` - generated plots

It is gitignored because it is large (~500MB+) and fully regenerable from the notebook.

## External API

The deck scoring and card data come from the MTG Assistant API at `https://mtg-assistant.up.railway.app`. Relevant endpoints:

- `/synergy/commander-profile` - archetype profile for a commander (cached per commander)
- `/synergy/analyze-deck` - composite score for a completed deck (terminal reward signal)

API calls are made once during setup and then cached. Training episodes run from cache with zero API calls.

## Running the Notebook

```bash
jupyter lab mtg_commander_dyna_q.ipynb
```

The notebook is self-contained. Run cells top to bottom to rebuild the cache, train, and evaluate. A fully-executed copy is saved as `mtg_commander_dyna_q_executed.ipynb`.
