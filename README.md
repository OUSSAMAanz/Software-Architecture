# Architectural Patterns Report
**Project:** Market Simulation + Multi-Agent RL System  
**Files Analyzed:** `C:\Users\diens\market_sim.py`, `D:\VSCODE\PyhtonStuff\`  
**Date:** 2026-04-02

---

## Project Overview

A multi-agent reinforcement learning system built on top of a custom economic market simulation engine. Agents trained with PPO (Proximal Policy Optimization) interact with a simulated medieval-to-industrial economy, either locally via a Gymnasium interface or remotely via a REST API server.

**Files:**

| File | Role |
|------|------|
| `market_sim.py` | Core simulation engine (~4200+ lines) |
| `train.py` | ML training pipeline (PPO via Stable-Baselines3) |
| `ServerTest.py` | FastAPI REST server for remote multi-agent play |
| `ClientTest.py` | Basic TCP client test |
| `NetworkTest.py` | Multi-threaded stress test (20 concurrent clients) |

---

## System Architecture

The project follows a clear **3-tier layered architecture**:

```
┌─────────────────────────────────────────────────────────────┐
│  ML Training Layer  (train.py)                              │
│  PPO training, Gymnasium wrapper, model checkpointing       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  API / Server Layer  (ServerTest.py)                        │
│  FastAPI REST endpoints, async multi-agent coordination     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Simulation Core  (market_sim.py)                           │
│  MarketEnvironment, Controllers, Buildings, Trade, Economy  │
└─────────────────────────────────────────────────────────────┘
```

---

## Pattern 1 — Layered Architecture

**Where:** Whole project  
**Files:** `market_sim.py` → `ServerTest.py` / `train.py`

Each layer has a single responsibility and only depends on the layer below it:

- **Simulation Core** is self-contained; no knowledge of HTTP or RL frameworks.
- **API Layer** imports `market_sim` and exposes it over REST; no training logic.
- **Training Layer** imports `market_sim` and wraps it for Stable-Baselines3; no server logic.

This separation means the simulation can be tested in isolation, served remotely, or trained against — all independently.

---

## Pattern 2 — Rich Domain Model

**Where:** `market_sim.py`  
**Classes:** `item`, `Building`, `ProductionChain`, `Market`, `MarketController`, `MarketEnvironment`

Objects encapsulate both **data and behavior** rather than being passive data containers. Example: `item` objects carry a `set[itemTag]` and `set[itemFlag]` and expose `hasTag()` / `hasFlag()` methods. `MarketController` (~1160 lines) owns buildings, treasury, trade relationships, and all agent-action logic.

**Class hierarchy** (flat — composition over inheritance):

```
MarketEnvironment
    ├── Market
    │     └── itemTrackUnit (per-item supply/demand tracking)
    ├── Infrastructure
    ├── PopClass  (population demographics)
    └── MarketController  (1..N agents)
          ├── Building  (1..N)
          │     └── ProductionChain  (1..N)
          │           └── productionRule
          └── TradeDeal  (pending/active trades)
```

---

## Pattern 3 — Adapter / Wrapper

**Where:** `train.py:48` (`MarketGymEnv`), `market_sim.py:4134` (`SingleAgentEnv`)

Multiple wrapper layers adapt the simulation to different consumers:

```
gym.Env  (Stable-Baselines3 interface)
    └── MarketGymEnv  (train.py)
            └── SingleAgentEnv  (market_sim.py)
                    └── MarketEnvironment  (market_sim.py)
```

| Wrapper | Adapts | To |
|---------|--------|----|
| `MarketGymEnv` | `SingleAgentEnv` / `MarketEnvironment` | `gym.Env` interface |
| `SingleAgentEnv` | `MarketEnvironment` | Simplified single-agent step/reset API |
| `VecNormalize` | `DummyVecEnv` / `SubprocVecEnv` | Running mean/std obs & reward normalization |

**Observation space:** `Box(530,)` float32 clipped to `[-1, 1]`  
**Action space:** `MultiDiscrete([13, 16, 8])` — 13 action types × 16 slots × 8 slots

---

## Pattern 4 — Factory

**Where:** `train.py:98` (`make_env()`), `market_sim.py:44` (`ACTION_SPECS`)

**`make_env()` — closure-based environment factory:**

```python
def make_env(agent_idx=0, seed=0):
    def _init():
        env = MarketGymEnv(agent_idx=agent_idx)
        env = Monitor(env)
        return env
    return _init
```

Called by vectorized env builders (`DummyVecEnv`, `SubprocVecEnv`) to spawn multiple independent environment instances.

**`ACTION_SPECS` — declarative schema registry:**

```python
ACTION_SPECS: dict[ActionType, list[ActionParamSpec]] = {
    ActionType.BUILD_BUILDING: [
        ActionParamSpec("building_type", "BuildingType", ...),
        ActionParamSpec("chains", "list", ...),
    ],
    ...
}
```

Each `ActionType` maps to its parameter schema. Acts as a metadata factory for RL action encoding/decoding.

---

## Pattern 5 — Synchronization Barrier

**Where:** `ServerTest.py` — `/step` endpoint

Multi-agent step coordination built from `asyncio.Lock` + `asyncio.Event`:

```
Agent 1 → POST /step (action[1])
    locked → pending[1] = action[1]
    1/N agents ready → asyncio.Event.wait(timeout=10s)

Agent 2 → POST /step (action[2])
    locked → pending[2] = action[2]
    N/N agents ready → env.step(all pending)
                     → Event.set()  (wake all waiters)
                     → Event.clear() (reset for next tick)

Both agents unblock → each receives their own StepResponse
```

A 10-second timeout prevents deadlock if an agent disconnects mid-episode. This pattern ensures all agents in a session always step the shared environment together.

---

## Pattern 6 — Strategy (CLI Mode Selection)

**Where:** `train.py` — argparse `--mode` flag

Pluggable execution strategies selected at runtime:

| Mode | Function | Description |
|------|----------|-------------|
| `train` | `train()` | Single or parallelized single-agent PPO training |
| `train_multi` | `train_multi()` | Multi-agent training (scaled `n_envs`) |
| `infer` | `run_inference()` | Model evaluation; optional CSV export of per-tick metrics |

Each strategy reuses the same environment and model infrastructure but differs in how episodes are collected and how results are consumed.

---

## Pattern 7 — Observer / Callback

**Where:** `train.py` — Stable-Baselines3 callbacks

Callbacks hook into the training loop as event listeners without coupling training logic to checkpoint/eval concerns:

| Callback | Trigger | Action |
|----------|---------|--------|
| `CheckpointCallback` | Every N timesteps | Save model `.zip` to `checkpoints/` |
| `EvalCallback` | Every N timesteps | Run evaluation episodes; save best model |

Both write to disk independently; the PPO loop has no knowledge of their implementation.

---

## Pattern 8 — Vectorized Environment

**Where:** `train.py:119`

Parallelizes environment instances for faster rollout collection:

```
SubprocVecEnv  — true multiprocessing (one OS process per env, CPU parallelism)
DummyVecEnv   — single-process sequential fallback (safer, easier to debug)
    └── both wrapped by VecNormalize
            norm_obs=True,    clip_obs=10.0
            norm_reward=True, clip_reward=10.0
```

`SubprocVecEnv` is used for production training; `DummyVecEnv` is the fallback when subprocess spawning is unavailable (e.g., Windows interactive sessions).

---

## Pattern 9 — DTO + Schema Validation

**Where:** `ServerTest.py` — Pydantic models

All data crossing the HTTP boundary is validated through Pydantic `BaseModel` DTOs:

```
BaseModel
    ├── ResetRequest    { num_agents: int, max_ticks: int, seed: Optional[int] }
    ├── ResetResponse   { session_id: str, obs: dict, state_dim: int, action_nvec: list }
    ├── StepRequest     { session_id: str, ctrl_id: int, action: list[int] }
    └── StepResponse    { session_id: str, ctrl_id: int, obs, reward, terminated,
                          truncated, tick: int, info: dict }
```

Type coercion and validation happen automatically at the FastAPI layer before any simulation logic is touched.

---

## Pattern 10 — Type Object (Enum Tag System)

**Where:** `market_sim.py` — `itemTag`, `itemFlag`, `ActionType`

Rather than a deep class hierarchy for items, the simulation uses a **set-based tag system** where each `item` carries multiple runtime type descriptors:

```python
class itemTag(Enum):   # 37 values — item categories
    food, crops, metal, alloy, fabric, luxury, explosive, ...

class itemFlag(Enum):  # 25 values — item properties
    isNutritional, isDurable, isValuable, isHardened, isFlammable, ...
```

**Multi-tag semantics:**
- `steel` → `{metal, alloy}` — satisfies both `metal` and `alloy` recipe slots
- `coal` → `{fuel, coal}` — satisfies generic `fuel` but also coal-specific recipes (coking, steam engines)
- `silk fabric` → `{fabric, luxury}` — satisfies `fabric` slots and triggers luxury bonuses

Production rules match inputs by **tag intersection**, enabling flexible recipe composition without combinatorial subclasses.

---

## Pattern Summary

| # | Pattern | Location | Purpose |
|---|---------|----------|---------|
| 1 | Layered Architecture | Whole project | Separation of simulation, serving, and training |
| 2 | Rich Domain Model | `market_sim.py` | Encapsulate economy logic in cohesive objects |
| 3 | Adapter / Wrapper | `MarketGymEnv`, `SingleAgentEnv`, `VecNormalize` | Bridge simulation to RL framework interfaces |
| 4 | Factory | `make_env()`, `ACTION_SPECS` | Controlled environment instantiation and action schema |
| 5 | Synchronization Barrier | `ServerTest.py /step` | Lock-free multi-agent tick coordination |
| 6 | Strategy | `train.py --mode` | Pluggable train / infer execution paths |
| 7 | Observer / Callback | SB3 callbacks | Decouple checkpoint and eval from training loop |
| 8 | Vectorized Environment | `DummyVecEnv` / `SubprocVecEnv` | Parallel rollout collection |
| 9 | DTO + Schema Validation | Pydantic models | Type-safe HTTP boundary enforcement |
| 10 | Type Object (Enum tags) | `itemTag`, `itemFlag` | Flexible multi-tag item classification |

---

## Key Dependencies

| Package | Used In | Role |
|---------|---------|------|
| `stable-baselines3` | `train.py` | PPO algorithm implementation |
| `gymnasium` | `train.py` | Standard RL environment interface |
| `fastapi` | `ServerTest.py` | REST API framework |
| `pydantic` | `ServerTest.py` | Request/response schema validation |
| `pyngrok` | `ServerTest.py` | Public tunnel to local server |
| `uvicorn` | `ServerTest.py` | ASGI server |
| `numpy` | `train.py`, `market_sim.py` | Numerical arrays for observations |
| `tensorboard` | `train.py` | Training curve visualization |
