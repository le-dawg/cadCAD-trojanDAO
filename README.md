
# cadCAD x TrojanDAO

This repository contains a cadCAD model prototype for a TrojanDAO-style token economy.  
The implementation lives in `CADtrojanDAO.ipynb` and focuses on token-holder balance dynamics under stochastic user actions and protocol tax rules.

## Modeling Approach

The model is built in cadCAD's standard structure:

1. `initial_conditions`: state variables and their starting values.
2. Policy functions: event logic that decides what happens at each timestep.
3. State update functions: deterministic accounting transformations applied after policy outputs.
4. `partial_state_update_blocks`: orchestration of which policies and update functions run together.
5. `simulation_parameters`: timeline and Monte Carlo controls.

### 1. State Representation

The main system state is:

- `token_holders`: vector of balances (index `0` is DAO pool, index `1` is guildbank, others are users).
- `BC_reserve`: reserve tracked in ETH-like units.
- `total_tokens`: aggregate token supply used for proportional updates.
- Tax and action parameters:
`amount_to_mint`, `amount_to_burn`, `DAO_tax_rate`, `redist_tax_rate`, `burn_tax_rate`.
- Indexing variables:
`n`, `DAO_pool_index`, `guildbank_index`, `update_index`.
- Temporary redistribution bucket:
`redist`.

This is a compact accounting model: balances and reserve are updated directly each timestep based on one selected actor and the configured rates.

### 2. Stochastic Interaction Layer (Policy)

`choose_holder` is the active policy in the configured run path.  
At each timestep it randomly samples a user index from `2..n-1` and returns it as `update_index`.

This is the model's non-deterministic component: different random draws produce different trajectories across runs.

### 3. Accounting Transitions (State Updates)

The notebook defines both mint and burn update functions, including:

- reserve updates (`update_BC_reserve_mint`, `update_BC_reserve_burn`)
- holder balance updates (`update_token_holders_mint`, `update_token_holders_burn`)
- supply updates (`update_total_tokens_mint`, `update_total_tokens_burn`)
- redistribution amount updates (`update_redistribution_amount_mint`, `update_redistribution_amount_burn`)
- proportional redistribution application (`redistribute`)

In the currently active `partial_state_update_blocks`, the burn path is wired:

- selected holder burns `amount_to_burn * holder_balance`
- holder balance decreases by burned amount
- DAO pool receives burn tax (`burn_tax_rate`)
- reserve decreases by burned amount net of burn tax
- total supply decreases by burned amount

Mint and redistribution functions are present for scenario extension, but are not all active in the final configured block.

### 4. cadCAD Execution Semantics

cadCAD executes timesteps according to:

- `T`: number of timesteps (`range(100)` in the active TrojanDAO setup)
- `N`: number of Monte Carlo runs (`1` in the active TrojanDAO setup)
- `M`: parameter sweep dictionary (empty in this notebook)

The notebook packages the model with:

- `Configuration(...)`
- `ExecutionMode` / `ExecutionContext`
- `Executor(...).execute()`

Output is converted into a pandas DataFrame (`raw_result -> df`) for analysis and plotting.

### 5. What This Prototype Is Good For

This model is useful for:

- stress-testing tax-rate assumptions against stochastic user behavior
- observing reserve/supply/balance trajectories over time
- extending toward richer policy logic (e.g., conditional minting, governance actions, parameter sweeps)

It is intentionally minimal and notebook-centric, making it easier to iterate on mechanism logic before turning it into a package/module.

## How To Run

### 1. Environment Setup

Requires Python 3.

Install dependencies:

```bash
pip install cadCAD numpy pandas matplotlib jupyter
```

### 2. Launch the Notebook

From repository root:

```bash
jupyter notebook CADtrojanDAO.ipynb
```

### 3. Execute the TrojanDAO Model Cells

Run cells from top to bottom through the section that:

- defines `initial_conditions`
- defines policy and state update functions
- defines `partial_state_update_blocks`
- builds `Configuration` and executes `Executor`

Then inspect `df` and plotting cells for trajectory analysis.

### 4. Optional Monte Carlo Extension

To run multiple stochastic trajectories, change:

```python
simulation_parameters = {
    "T": range(100),
    "N": 50,
    "M": {}
}
```

and re-run configuration + execution cells.

## Notes

- The notebook also contains legacy tutorial sections unrelated to TrojanDAO (robot/marbles examples).
- Keep analysis focused on the TrojanDAO state and update blocks for reproducible results in this repository.
