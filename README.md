# Dyna-Q Reinforcement Learning for MTG Commander Deck Construction

Penn State MSAI reinforcement learning course project. A model-based Dyna-Q agent that learns to construct Magic: The Gathering Commander decks from a real 4,393-card Archidekt collection across 12 legendary commanders.

**Author:** Keith Coughlin (kac7102@psu.edu)

## Research Question

Can a model-based RL agent learn a generalizable Commander deck-building policy from a constrained personal card collection, using only structural signals (role balance, mana curve, synergy density) as reward?

## Approach

- **Agent:** Tabular Dyna-Q with epsilon-greedy policy (epsilon 0.15 to 0.01, decay 0.995), gamma 0.95, alpha 0.1, and n = 15 planning steps per real transition
- **Environment:** `gymnasium.Env` that samples one of 12 commanders per episode and fills 63 card slots one at a time. Basic lands are auto-added after to reach 100 cards
- **State:** 22-element vector encoding role counts (ramp, removal, card draw, creatures, board wipes), mana curve distribution across CMC buckets 1 to 6+, color pip distribution, episode progress, and a one-hot of the commander's color identity
- **Action space:** Individual cards from the commander's color-legal pool. Already-picked cards are masked (singleton constraint)
- **Reward:** Intermediate shaping rewards on every step (role-gap filling, curve penalties) plus a terminal reward computed locally from role balance, synergy density against the commander's cached profile, mana curve quality, and card quality (EDHREC rank)
- **Training:** 1,000 episodes in roughly 75 seconds on a laptop. Policy stabilizes by episode 500 (Jaccard similarity of top-20 card lists)

The original Week 1 formulation proposed a category-abstraction design with 729 binned states and ~50 category actions. The final implementation uses individual cards as actions and a denser 22-element state representation. See [docs/week1_rl_formulation.md](docs/week1_rl_formulation.md) for the original formulation and [deliverables/Keith_Coughlin_RL_Project_Final.docx](deliverables/Keith_Coughlin_RL_Project_Final.docx) for the full final report with results.

## Results Summary

Greedy evaluation across 48 episodes (4 per commander):

- **Mean terminal reward:** -0.205 (standard deviation 0.176)
- **Best:** Korvold, Fae-Cursed King (+0.111, sacrifice archetype correctly detected by API)
- **Worst:** Atraxa, Praetors' Voice (-0.468, four-color pool of 3,232 cards is too diffuse for tabular methods)
- **Policy stability:** 10 of 12 commanders hit perfect 1.0 Jaccard similarity between episode 500 and episode 1,000

See the final report for the full per-commander table, role coverage analysis, and discussion.

## Repository Layout

```
.
├── README.md
├── mtg_commander_dyna_q.ipynb             # Main notebook (all modules + training + eval)
├── docs/
│   └── week1_rl_formulation.md            # Week 1 formulation (superseded by final impl)
├── exports/                               # Rendered copies of the notebook
│   ├── mtg_commander_dyna_q.html
│   ├── mtg_commander_dyna_q.pdf
│   └── mtg_commander_dyna_q_executed.ipynb
├── deliverables/                          # Course-submission .docx files
│   ├── Keith_Coughlin_RL_Project_Final.docx   # Final submission report
│   ├── Reinforcement_Learning_Project_Template.docx
│   ├── Reinforcement_Learning_Project_Week1.docx
│   ├── Assignment_4_Results.docx
│   ├── Assignment_4_5_Results.docx
│   └── RL_Project_Writeup.docx
├── scripts/
│   └── populate_template.py               # Helper to populate the course .docx template
└── cache/                                 # (gitignored) collection, checkpoints, plots
```

The notebook reads cached data from a hard-coded `./cache` relative path, so run Jupyter with the repo root as your working directory.

## Cache Directory

The `cache/` directory holds data that is slow to regenerate but not worth versioning:

- `scryfall_bulk.pkl`, `collection_enriched.pkl` - cached Scryfall metadata and the enriched Archidekt collection (4,393 cards)
- `commander_profile_*.json` - per-commander synergy profiles fetched from the API
- `checkpoint.pkl`, `eval_results.pkl` - trained Q-table and evaluation results
- `metric1_learning_curve.png` through `metric5_policy_stability.png`, `api_validation.png` - generated plots

It is gitignored because it is around 550MB (the trained checkpoint alone is 542MB) and fully regenerable from the notebook.

## External API

Card data and post-training validation come from the MTG Assistant API at `https://mtg-assistant.up.railway.app`, a personal project hosted on Railway. Relevant endpoints:

- `/collections/ingest-url-stream` - ingests the Archidekt collection at session start (once, cached)
- `/synergy/commander-profile` - fetches the synergy card list for each commander (once per commander, cached)
- `/decks/analyze-complete` - full synergy, bracket, and engine analysis. Used only for post-training validation, not during the training loop

Training episodes compute terminal rewards locally from cached card metadata. The original plan was to call `/decks/analyze-complete` every episode, but the 3 to 5 second per-call latency would have stretched training to 60 to 90 minutes. The local four-component reward correlates at 0.968 with the API's synergy score on validation decks.

## Running the Notebook

```bash
jupyter lab mtg_commander_dyna_q.ipynb
```

The notebook is self-contained. Run cells top to bottom to rebuild the cache, train, and evaluate. A fully-executed copy with all outputs intact lives at `exports/mtg_commander_dyna_q_executed.ipynb`.
