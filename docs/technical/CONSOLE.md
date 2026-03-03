# In-Game Console

An in-game command line interface, always available (including release builds). Opened with the `~` key (configurable). This is a power user tool for experimentation, debugging, modding, and learning.

## Design Principles

1. **Always available** — no debug-only builds. Single player game, no reason to restrict access.
2. **Full access** — can read and write every variable in the simulation. No artificial limits.
3. **SFC-aware** — write commands that modify money stocks do so through proper double-entry operations, maintaining consistency. The console doesn't bypass the economic model — it operates through it.
4. **Scriptable** — supports running script files with sequences of commands for repeatable setups.
5. **Discoverable** — `help` command, tab-completion, and clear error messages.
6. **Mod-extensible** — mods can register new console commands through the plugin API.

## Interface

```
┌─────────────────────────────────────────────────────────────┐
│ > set government.spending 5000                              │
│ Government spending set to 5000/month                       │
│ > query households.low_income.balance                       │
│ Households (Low Income) deposit balance: 12,450             │
│ > help spending                                             │
│ Available spending commands:                                │
│   set government.spending <amount>                          │
│   set government.allocation.infrastructure <percent>        │
│   set government.allocation.services <percent>              │
│   set government.allocation.transfers <percent>             │
│ >                                                           │
└─────────────────────────────────────────────────────────────┘
```

- Overlay at the bottom or top of the screen
- Semi-transparent background
- Scrollable output history
- Toggle open/close with `~` (configurable)

## Command Categories

### Inspection (Read)

Query any value in the simulation.

```
query <path>                          # Query a specific value
query government.balance              # Treasury balance
query government.bonds.outstanding    # Total bonds outstanding
query households.low_income.balance   # Low income deposit balance
query households.*.balance            # All household class balances
query firms.agriculture.employees     # Agriculture sector employment
query firms.*.price                   # All sector prices
query banks.reserves                  # Bank reserve balance
query banks.loans.total               # Total outstanding loans
query economy.employment_rate         # Current employment rate
query economy.inflation_rate          # Current inflation rate
query economy.gdp                     # Current GDP
query economy.indicators              # All key indicators at once
query sfc.check                       # Run SFC consistency check, show any imbalances
```

**Wildcards:** `*` matches any single segment. `**` matches multiple segments.

```
query firms.**                        # All firm data (deep)
query **.balance                      # All balances everywhere
```

### Manipulation (Write)

Modify simulation state. All monetary operations go through proper SFC channels.

**Government:**
```
set government.spending <amount>                  # Set monthly spending
set government.allocation.infrastructure <pct>    # Set allocation %
set government.allocation.services <pct>
set government.allocation.transfers <pct>
set government.tax_rate <rate>                    # Set tax rate (0.0-1.0)
spend <amount>                                    # One-time government spend
tax <amount>                                      # One-time tax collection
issue_bonds <amount>                              # Force bond issuance
```

**Central Bank:**
```
set cb.policy_rate <rate>             # Set central bank policy rate
cb.buy_bonds <amount>                 # Central bank buys bonds (QE)
cb.sell_bonds <amount>                # Central bank sells bonds (QT)
```

**Economy:**
```
set firms.<sector>.markup <value>     # Set sector markup
set firms.<sector>.productivity <value>
set firms.<sector>.wage <value>       # Set posted wage
set households.<class>.balance <amount>   # Set deposit balance (via government transfer to maintain SFC)
add households.<class>.balance <amount>   # Add to balance (via government transfer)
set economy.employment_rate <rate>    # Force employment rate (adjusts hiring)
```

**Agents:**
```
spawn firm <sector>                   # Create a new firm
spawn household <class> <count>       # Create households
remove firm <sector>                  # Remove a firm
```

### Time Control

```
pause                                 # Pause simulation
resume                                # Resume simulation
speed <multiplier>                    # Set speed (1, 2, 5, 10, 50, 100)
tick                                  # Advance exactly one tick (while paused)
tick <count>                          # Advance N ticks
skip <months>                         # Skip forward N months
```

### Scenario / Game State

```
scenario list                         # List available scenarios
scenario load <name>                  # Load a scenario
scenario restart                      # Restart current scenario
save <name>                           # Save game state
load <name>                           # Load game state
reset                                 # Reset to initial state
```

### Information

```
help                                  # List all command categories
help <command>                        # Detailed help for a command
help <category>                       # List commands in a category
history                               # Show command history
history clear                         # Clear command history
commands                              # List all registered commands (including mod commands)
version                               # Show game and mod versions
```

### Balance Sheet / SFC

```
balance government                    # Show government balance sheet
balance banks                         # Show banking sector balance sheet
balance households                    # Show household sector balance sheet
balance households.low_income         # Show specific class balance sheet
balance firms                         # Show firm sector balance sheet
balance firms.agriculture             # Show specific sector balance sheet
balance all                           # Show all balance sheets
sfc check                             # Verify SFC consistency (all balances sum to zero)
sfc flows                             # Show last tick's flow of funds
sfc matrix                            # Show the transaction flow matrix
```

### Charts / Logging

```
log <path> [duration]                 # Log a variable to console output each tick
log economy.inflation_rate 12         # Log inflation for 12 ticks
log stop                              # Stop all logging
export <path> <filename>              # Export variable history to CSV
export economy.* data.csv             # Export all economy indicators
```

## Scripting

### Running Script Files

```
run <filename>                        # Execute commands from a script file
run scripts/test-setup.txt            # Run a test setup script
```

Script files contain one command per line. Empty lines and lines starting with `#` are ignored.

### Example Script: `scripts/high-spending-test.txt`

```
# Setup: test high government spending scenario
pause
reset

# Set initial conditions
set government.spending 10000
set government.allocation.infrastructure 0.4
set government.allocation.services 0.3
set government.allocation.transfers 0.3
set government.tax_rate 0.20

# Run and observe
log economy.inflation_rate
log economy.employment_rate
log economy.gdp
resume
speed 5
```

### Script Locations

- `data/base/scripts/` — base game scripts
- `mods/<mod-name>/scripts/` — mod-provided scripts
- `user/scripts/` — user-created scripts

## SFC-Aware Write Operations

Write commands that affect money stocks do NOT directly edit numbers. They operate through the economic model to maintain SFC consistency:

| Command | Actual operation |
|---|---|
| `spend 1000` | Government spending of 1000 → creates reserves → creates deposits in recipient accounts |
| `add households.low_income.balance 500` | Government transfer of 500 to low income households → proper spending flow |
| `set households.low_income.balance 10000` | Calculates difference from current balance, executes government transfer for the delta |
| `tax 2000` | Forces tax collection of 2000 → destroys deposits → destroys reserves |

**Exception:** the `set` command with the `--force` flag bypasses SFC and directly edits values. This will trigger an SFC consistency warning and is intended only for debugging:

```
set --force banks.reserves 99999     # UNSAFE: directly sets reserves, breaks SFC
sfc check                            # Will report imbalance
sfc repair                           # Attempts to restore consistency by adjusting government balance
```

## Mod-Extensible Commands

Mods can register custom commands through the plugin API (post-MVP):

```csharp
public class MyMod : IGameMod
{
    public void RegisterCommands(IConsoleRegistry registry)
    {
        registry.Register("stockmarket", new StockMarketCommand());
        registry.Register("ipo", new IPOCommand());
    }
}
```

Users can then type `stockmarket status` or `ipo create TechCorp 1000000` in the console.

## MVP Scope

For the MVP, the console includes:

**In MVP:**
- Console UI (toggle with `~`, text input, scrollable output)
- `query` commands for all simulation state
- `set` commands for government policy (spending, allocation, tax rate)
- `spend` and `tax` one-time operation commands
- `balance` commands for all sectors
- `sfc check` for consistency verification
- `pause`, `resume`, `speed`, `tick` time controls
- `help` system
- `run` for basic script execution
- Command history (up/down arrow)

**Not in MVP:**
- Tab completion
- Wildcard queries (`*`, `**`)
- Agent spawn/remove commands
- CSV export
- Mod command registration
- `--force` flag for unsafe writes
- `sfc repair`
