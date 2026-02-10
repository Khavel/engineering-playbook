# Parallel Engine Architecture

## Problem

You need to add a fundamentally different execution mode to a large, layered engine (~2000 lines, 5+ inheritance layers) without risking regression in the stable codebase. Direct modification means:

- Every layer must be touched, breaking the open/closed principle
- Testing burden multiplies (must re-validate all existing behavior)
- Merge conflicts with any in-flight feature work
- Rollback requires reverting scattered changes across many files

## Solution

Create a **parallel engine** in a new file that mirrors the inheritance hierarchy but implements the new behavior from scratch. The server provides a **dual-mode endpoint** that dispatches to either engine based on a `mode` parameter.

### Architecture

```
# Legacy (untouched)
engine.py: MovementSim → Phase3Sim → Phase4Sim → Phase5Sim → Phase6Sim

# New parallel engine
continuous_engine.py: ContinuousMovementSim → ContinuousLootPveSim →
                      ContinuousCombatSim → ContinuousExtractionSim →
                      ContinuousReplaySim
```

### Dual-Mode Endpoint

```python
@app.route('/api/simulate', methods=['POST'])
def simulate():
    data = request.get_json()
    mode = data.get('mode', 'continuous')  # new default, legacy fallback

    if mode == 'legacy':
        sim = LegacySimulator(scenario)
        result = sim.run()
    else:
        sim = NewSimulator(scenario, new_config)
        result = sim.run()

    return jsonify(result)
```

### Frontend Auto-Detection

```javascript
// Detect mode from data shape, not from config
const mode = keyframes.some(k => k.x !== undefined) ? 'continuous' : 'zone';

// Render differently per mode
if (mode === 'continuous') {
    drawContinuousMap(ctx, agents, mapConfig);
} else {
    drawZoneGrid(ctx, agents, zones);
}
```

## Key Points

- **Zero risk to existing code** — legacy engine file is never modified
- **Independent testing** — new engine has its own validation without affecting legacy tests
- **Gradual migration** — switch the default mode when confident, remove legacy later
- **Shared schemas** — both engines use the same Pydantic models and replay format, just populate different fields
- **Feature parity tracking** — inheritance layers in the new engine map 1:1 to legacy layers, making it clear what's been ported

## When to Use

- Adding a fundamentally different execution model (grid → continuous, sync → async)
- Legacy codebase is large (>1000 lines) with deep inheritance
- You need both modes to coexist during a transition period
- The existing codebase has no test coverage or limited tests (risky to modify)
- Multiple developers working on the same engine file

## When NOT to Use

- Small codebase where inline changes are manageable
- The change is additive (new feature, not a mode change)
- You can use the Strategy pattern within the existing class hierarchy
- The legacy mode will be immediately deprecated

## Trade-offs

| Aspect | Parallel Engine | In-Place Modification |
|--------|----------------|----------------------|
| Risk | Zero regression risk | High regression risk |
| Code duplication | Some shared logic duplicated | No duplication |
| Maintenance | Two codebases to maintain | One codebase |
| Migration path | Clean cutover when ready | Already migrated |
| Rollback | Trivial (delete new file) | Complex (revert many files) |

## Real-World Example

In ArcRaiders Sim, we needed to replace a zone-based 3×3 grid movement system with continuous 2D coordinates. The legacy `engine.py` was 1938 lines with 5 inheritance layers. Instead of modifying it, we created `continuous_engine.py` (797 lines) with a parallel 5-layer hierarchy. The server dispatches between them via a `mode` parameter, and the frontend auto-detects the mode from keyframe data shape.

## Prevention

When designing extensible engines, consider the **Strategy pattern** upfront:

```python
class SimulationEngine:
    def __init__(self, movement_strategy: MovementStrategy):
        self.movement = movement_strategy

    def tick(self):
        self.movement.update(self.agents)
```

This avoids the need for parallel engines entirely — but requires foresight about future execution modes.
