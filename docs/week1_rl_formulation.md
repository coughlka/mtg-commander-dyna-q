# Week 1: Research Goal & RL Problem Formulation

## Research Goal

**Can a model-based reinforcement learning agent learn to construct competitive Magic: The Gathering Commander decks from a constrained physical card collection, using only card category abstractions and coarse state representations?**

Commander is a 100-card singleton format where deck construction requires balancing dozens of competing concerns: mana curve, removal density, card advantage, synergy with the commander, and win condition coherence. Expert players rely on years of pattern recognition to navigate these trade-offs. This project asks whether a Dyna-Q agent — given a tractable abstraction of the deck-building problem — can discover effective construction policies that rival or approach human-level deck building.

The business/practical motivation is straightforward: most Commander players own large collections (3,000–8,000+ cards) and struggle to build optimized decks from what they own. An RL agent that learns to build strong decks from a specific collection — not from the entire card pool — provides personalized, actionable deck recommendations grounded in cards the player actually has.

### Why Dyna-Q?

The problem has two properties that make Dyna-Q a strong fit:

1. **Simulated experience is cheap.** Once card metadata is cached locally, the agent can simulate deck-building episodes with zero API cost. Dyna-Q's planning steps exploit this by replaying past transitions through a learned model, dramatically improving sample efficiency over pure Q-learning.

2. **The state-action space is tractable but not trivial.** With ~729 discretized states and ~50 card category actions, the Q-table has ~36,450 entries — large enough that pure exploration would be slow, but small enough that tabular methods converge reliably. Dyna-Q's model-based planning fills exactly this gap.

---

## RL Problem Formulation

### Agent

The agent is a **Dyna-Q learner** that builds a Commander deck one card category at a time. At each time step, it observes the current state of the partially-built deck and selects a card category to add next. A downstream heuristic then picks the best specific card from the player's collection for that category.

The agent maintains:
- A **Q-table** Q(s, a) mapping state-action pairs to expected cumulative reward
- A **model** Model(s, a) → (s', r) learned from experience, used for planning steps
- An **ε-greedy policy** for balancing exploration and exploitation

### Environment

The environment represents the **deck-building process** for a single commander, implemented as a `gymnasium.Env`.

**Episode structure:**
1. A commander is selected (fixed or randomly chosen from a curated roster of 10–15 commanders)
2. The agent fills all 99 deck slots one at a time (1 commander + 99 selected cards = 100 total). Every slot is a strategic decision, including lands — the agent chooses among non-basic land categories (shock lands, fetch lands, dual lands, utility lands, etc.) alongside spells and creatures.
3. The episode terminates when all 99 slots are filled
4. A terminal reward is computed via the `/synergy/analyze-deck` API endpoint

**Key environment rules (hard constraints):**
- **Singleton:** No duplicate card names may appear in the deck (except basic lands)
- **Color identity:** Every non-land card must fall within the commander's color identity
- **Collection-bound:** Only cards the player actually owns are available
- **Category availability:** If a category has no remaining eligible cards in the collection, that action is masked (unavailable)

The environment enforces all constraints — the agent never sees an illegal action as available.

### State Space

The state represents the **composition of the partially-built deck** using coarse bins. Rather than tracking individual cards (which would create an astronomically large state space), the state captures deck-level statistics that matter for construction decisions.

**State variables (each discretized into 3 bins: low / medium / high):**

| Variable | What It Captures | Why It Matters |
|---|---|---|
| **Mana curve position** | Average CMC of cards selected so far | Keeps the deck castable; too high = clunky, too low = runs out of gas |
| **Removal density** | Fraction of slots allocated to removal (spot + board wipes) | Decks need answers to threats; too few = vulnerable, too many = no proactive plan |
| **Card advantage density** | Fraction of slots allocated to draw/advantage engines | The primary resource axis in Commander; running out of cards = losing |
| **Synergy concentration** | How many picks align with the commander's primary archetype | Focused decks outperform "good stuff" piles; measures strategic coherence |
| **Ramp density** | Fraction of slots allocated to mana acceleration | Commander games are won by whoever deploys threats first |
| **Slot progress** | How far through the episode (early / mid / late) | What the deck needs changes as it fills up — ramp early, finishers late |

With 6 variables × 3 bins each = **729 possible states**. This is deliberately coarse — fine enough to distinguish meaningfully different deck states, small enough for tabular convergence.

### Action Space

Each action corresponds to a **card category** — an abstraction over individual cards. There are approximately **50 categories** covering the functional roles needed in a Commander deck:

| Category Group | Example Categories |
|---|---|
| **Lands** | land_fetch, land_shock, land_dual, land_triome, land_utility, land_mdfc, land_basic |
| **Mana** | ramp_creature, ramp_spell, mana_rock, mana_dork |
| **Removal** | spot_removal_creature, spot_removal_spell, board_wipe, artifact_removal, enchantment_removal |
| **Card Advantage** | card_draw_spell, card_draw_engine, impulse_draw, tutor, recursion |
| **Counterspells** | counterspell_hard, counterspell_soft |
| **Creatures** | creature_aggro, creature_utility, creature_evasive, creature_token_producer |
| **Protection** | protection_self, protection_permanent, hexproof_source |
| **Synergy** | tribal_payoff, tribal_lord, combo_piece, sacrifice_outlet, aristocrat_payoff |
| **Value Engines** | enchantment_value, artifact_value, planeswalker |
| **Finishers** | wincon_combat, wincon_combo, wincon_alternate |

When the agent picks a category, a **card selection heuristic** chooses the best available card from the collection within that category, considering:
- Synergy with the specific commander (keyword/archetype overlap)
- Standalone card power (CMC efficiency, effect breadth)
- What's already in the deck (avoiding redundancy)

This two-tier design (RL picks categories, heuristic picks cards) is what makes the problem tractable: ~50 actions instead of ~5,000.

### Reward Function

The reward function combines **intermediate shaping rewards** during the episode with a **terminal reward** at the end.

**Intermediate rewards (per step):**
- **Mana curve bonus/penalty:** Small positive reward for picks that keep average CMC in a healthy range (2.5–3.5), small penalty for pushing it too high or too low
- **Diversity bonus:** Small reward for picking a category that addresses an underrepresented deck need (e.g., if removal density is low and the agent picks removal)
- **Synergy bonus:** Small reward for picks that align with the commander's archetype profile (obtained from `/synergy/commander-profile` at setup, cached)
- **Redundancy penalty:** Small penalty for picking a category that's already well-represented, discouraging lopsided decks

These shaping rewards are kept small relative to the terminal reward to avoid overwhelming the true objective.

**Terminal reward (end of episode):**
- Computed by calling `/synergy/analyze-deck` on the completed deck
- Incorporates the API's composite score: synergy rating, power heuristics (fast mana density, interaction density, card advantage density, average CMC, combo potential), and IPOM role balance
- This is the ground truth signal — it evaluates the complete deck holistically

The terminal reward is the primary learning signal. Intermediate rewards provide gradient to guide early learning when the terminal signal is too sparse.

### Transition Dynamics

The transitions are **deterministic given the agent's action and the heuristic's card selection:**

1. Agent observes state s_t (binned deck composition)
2. Agent selects action a_t (card category)
3. Heuristic selects best available card from that category
4. Card is added to deck; environment computes new state s_{t+1}
5. Intermediate reward r_t is computed
6. If all meaningful slots are filled → episode ends, terminal reward is computed

The **Dyna-Q model** learns these transitions: Model(s, a) → (s', r). Because the environment is largely deterministic (the heuristic's card choice is predictable given the deck state), the model is highly accurate, and planning steps are very effective.

### Summary Table

| RL Component | This Project |
|---|---|
| **Agent** | Dyna-Q with ε-greedy policy, tabular Q-table, learned transition model |
| **Environment** | Sequential deck builder (gymnasium.Env); 99 card slots per episode (all strategic, including lands) |
| **State** | 6 deck composition metrics × 3 bins = 729 states |
| **Actions** | ~50 card categories (functional roles); heuristic maps to specific cards |
| **Reward** | Intermediate shaping (curve, diversity, synergy) + terminal API-based deck score |
| **Model** | Tabular Model(s,a)→(s',r) enabling n planning steps per real step |
| **Constraints** | Singleton, color identity, collection-bound — enforced via action masking |
| **Training** | Zero API calls during episodes; terminal reward via /synergy/analyze-deck |

### Scalability Justification

| Approach | State Space | Action Space | Q-Table Size | Feasibility |
|---|---|---|---|---|
| Individual cards, continuous state | ~10^24 | 3,000–8,000 | Intractable | No |
| Individual cards, binned state | ~729 | 3,000–8,000 | ~5.8M | Borderline |
| **Card categories, binned state** | **~729** | **~50** | **~36,450** | **Yes** |

The category abstraction reduces the problem by ~5 orders of magnitude while preserving the strategic decisions that matter for deck quality.
