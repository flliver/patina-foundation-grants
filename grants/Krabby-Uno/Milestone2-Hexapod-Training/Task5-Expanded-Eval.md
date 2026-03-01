# Task 5 — EXPANDED EVALUATION SUITE

Goal: Add 10+ diverse evaluation scenarios. Create framework for identifying model limitations through varied real-world scenarios.

**Terminology:**
- **Scene**: Static USD environment (e.g., `backyard.usd`)
- **Scenario**: Scene + robot + mission parameters (start/goal positions, time limits, etc.)

Outputs
- 10+ new USD scenes
- 10+ evaluation scenarios using those scenes
- Updated `make eval` target for full suite
- Registry system for adding new scenarios
- Evaluation results across all scenarios
- Analysis of failure modes and limitations

Acceptance Criteria
- 10+ diverse scenarios added and documented
- `make eval` runs full suite headless
- Results stored per-scenario with success metrics
- Failure analysis identifies specific limitations
- AI can add new scenario to registry and include in eval
- **At least one scenario** derived from a 3D-from-images pipeline (e.g. SAM 3D or similar: real-world images → 3D mesh → walkable USD).
- **Final validation**: Spin up the robot in simulation and drive it with an **input controller (joystick)** on at least one of the 3D meshes (e.g. SAM-3D–derived or other generated mesh). Document the command and capture a short clip or log.

---

**NOTE**: All code examples in this document are AI-generated pseudocode with human comments where appropriate. Treat them as design guidance, not production-ready code. Actual implementation may require adjustments based on Isaac Sim API versions and specific requirements.

---

## 1. Evaluation Suite Overview

### Scenario Categories
```python
scenario_categories = {
    'residential': [
        'backyard_traverse',
        'home_side_passage',
        'deck_navigation',
    ],
    'obstacles': [
        'junk_pile_avoid',
        'debris_navigate',
    ],
    'terrain': [
        'uneven_lawn_cross',
        'gravel_path_follow',
    ],
    'navigation': [
        'car_circumnavigate',
        'gate_passage',
    ],
    'stairs': [
        'stairs_3step_climb',
        'deck_stairs_ascend',
    ],
}
```

### Success Criteria Per Scenario
Not all scenarios need to pass - goal is to identify limitations:
- **Must pass (>80% success)**: Flat yards, simple obstacles
- **Should pass (>60% success)**: Uneven terrain, navigation around objects
- **May fail (<60% success)**: Stairs, tight spaces, complex debris

---

## 1.5 Scenario generation from real-world meshes

Add **backyard (or real-world) scenario generation** using a 3D-from-images pipeline so at least one evaluation scenario is derived from real-world imagery and usable as walkable terrain.

### Tooling options
- **SAM 3D** (Meta): Single-image (or few-image) 3D reconstruction; outputs textured 3D mesh. Post-process (scale, collision, export) to produce USD or collision mesh for Isaac Sim. See Meta SAM 3D documentation.
- **Messy.ai** (or similar): If it produces 3D meshes from 2D pictures, use as an option; document the workflow (images → mesh → USD).
- Other tools that convert street-level or Street View–style images to 3D mesh can be used; document the pipeline.

### Workflow
1. **Input**: Street View (or similar) images of a backyard, yard, or outdoor area.
2. **Reconstruction**: Run SAM 3D (or chosen tool) to produce a 3D mesh (geometry + optional texture).
3. **Post-process**: Clean, scale, and export as USD (or collision mesh) compatible with Isaac Sim. Ensure the mesh is walkable (e.g. ground plane, no inverted normals).
4. **Integration**: Add the scene to the scenario registry; use it in `make eval` and in the final joystick validation below.

### Deliverable
At least one scenario whose scene USD was generated from real-world images via this (or a documented equivalent) pipeline. Document the exact tool, commands, and steps.

---

## 2. Scenario Registry System

### Registry Format
```yaml
# assets/eval_scenarios/scenario_registry.yaml

scenarios:
  - name: backyard_traverse
    scene_usd: assets/eval_scenes/residential/home_yard_01.usd  # Static scene
    robot_usd: assets/crab_hex.usd                              # Robot to use
    category: residential
    difficulty: easy
    start_pos: [0.0, 0.0, 0.5]      # Mission start
    goal_pos: [10.0, 0.0, 0.5]      # Mission goal
    success_criteria:
      min_progress: 8.0              # Mission parameters
      max_time: 30.0
      max_falls: 0
    description: "Traverse flat backyard with grass"
```

### Registry Loader
```python
# eval_registry.py
import yaml
from dataclasses import dataclass

@dataclass
class EvalScenarioCfg:
    """Evaluation scenario: scene + robot + mission parameters."""
    name: str
    scene_usd: str          # Static environment
    robot_usd: str          # Robot to evaluate
    category: str
    difficulty: str
    start_pos: list         # Mission start
    goal_pos: list          # Mission goal
    success_criteria: dict  # Mission parameters
    description: str

class EvalRegistry:
    def __init__(self, registry_path: str):
        with open(registry_path, 'r') as f:
            data = yaml.safe_load(f)
        self.scenarios = [EvalScenarioCfg(**s) for s in data['scenarios']]
```

---

## 3. Make Target Integration

```makefile
# Evaluate all scenarios
eval-full:
\tpython scripts/evaluate_all_scenarios.py \
\t\t--model logs/rsl_rl/crab_student_baseline/v1/model_15000.pt \
\t\t--registry assets/eval_scenarios/scenario_registry.yaml \
\t\t--num_episodes 10 \
\t\t--headless

# Evaluate single scenario
eval-scenario:
\tpython scripts/evaluate_scenario.py \
\t\t--model logs/rsl_rl/crab_student_baseline/v1/model_15000.pt \
\t\t--scenario $(SCENARIO) \
\t\t--num_episodes 10
```

---

## 4. Final validation: joystick-on-mesh

**Requirement**: As final acceptance for this task, spin up the robot in simulation and drive it with an **input controller (joystick)** on at least one of the 3D meshes (e.g. a SAM-3D–derived or other generated mesh from Section 1.5).

### Steps
1. Load the chosen mesh scene in Isaac Sim / Isaac Lab (same stack as training).
2. Spawn the crab robot in the scene.
3. Connect a gamepad/joystick and use the existing joystick control path (e.g. from Milestone 6 or equivalent) to drive the robot.
4. Document the exact command(s) to launch sim + scene + joystick control.
5. Capture a short clip or log showing the robot moving under joystick control on the mesh.

This validates the full loop: generated real-world terrain → sim → human-in-the-loop control.

---

## 5. Success Criteria Checklist

- [ ] 10+ USD scenes created/sourced
- [ ] 10+ scenarios defined in registry
- [ ] At least one scenario from 3D-from-images pipeline (SAM 3D or similar)
- [ ] Scenario registry YAML complete
- [ ] `make eval-full` runs all scenarios
- [ ] Results stored as JSON per scenario
- [ ] Failure analysis identifies limitations
- [ ] AI can add new scenario using Python library
- [ ] Joystick-on-mesh demo completed and documented (command + clip/log)

---

## 6. Adding New Scenarios (AI Guide)

### Python Library for Scenario Management

```python
# scripts/scenario_registry_manager.py
"""Helper library for managing evaluation scenario registry."""

import yaml
from pathlib import Path
from typing import List

class ScenarioRegistryManager:
    """Manage evaluation scenario registry."""
    
    def __init__(self, registry_path: str = "assets/eval_scenarios/scenario_registry.yaml"):
        self.registry_path = Path(registry_path)
        self.registry_path.parent.mkdir(parents=True, exist_ok=True)
        
        if self.registry_path.exists():
            with open(self.registry_path, 'r') as f:
                self.data = yaml.safe_load(f) or {'scenarios': []}
        else:
            self.data = {'scenarios': []}
    
    def add_scenario(
        self,
        name: str,
        scene_usd: str,
        robot_usd: str = "assets/crab_hex.usd",
        category: str,
        start_pos: List[float],
        goal_pos: List[float],
        difficulty: str = "medium",
        min_progress: float = None,
        max_time: float = 40.0,
        max_falls: int = 1,
        description: str = ""
    ):
        """Add a new scenario to the registry.
        
        Args:
            name: Unique scenario identifier (snake_case)
            scene_usd: Path to scene USD file (static environment)
            robot_usd: Path to robot USD file
            category: One of: residential, obstacles, terrain, navigation, stairs
            start_pos: [x, y, z] robot spawn position
            goal_pos: [x, y, z] target position
            difficulty: easy, medium, hard, very_hard
            min_progress: Minimum distance (meters), defaults to 80% of goal distance
            max_time: Time limit (seconds)
            max_falls: Maximum falls before failure
            description: Human-readable description
        """
        # Calculate default min_progress if not provided
        if min_progress is None:
            import math
            distance = math.sqrt(
                (goal_pos[0] - start_pos[0])**2 + 
                (goal_pos[1] - start_pos[1])**2
            )
            min_progress = distance * 0.8
        
        # Check if scenario already exists
        existing_names = [s['name'] for s in self.data['scenarios']]
        if name in existing_names:
            raise ValueError(f"Scenario '{name}' already exists in registry")
        
        # Create scenario entry
        scenario = {
            'name': name,
            'scene_usd': scene_usd,
            'robot_usd': robot_usd,
            'category': category,
            'difficulty': difficulty,
            'start_pos': start_pos,
            'goal_pos': goal_pos,
            'success_criteria': {
                'min_progress': min_progress,
                'max_time': max_time,
                'max_falls': max_falls
            },
            'description': description
        }
        
        self.data['scenarios'].append(scenario)
        self._save()
        print(f"✓ Added scenario '{name}' to registry")
    
    def _save(self):
        """Save registry to YAML file."""
        with open(self.registry_path, 'w') as f:
            yaml.dump(self.data, f, default_flow_style=False, sort_keys=False)
```

### Example Script: Add a New Scenario

```python
# scripts/add_eval_scenario.py
"""Example script to add a new evaluation scenario."""

from scenario_registry_manager import ScenarioRegistryManager

# Create manager
manager = ScenarioRegistryManager()

# Add a new residential scenario
manager.add_scenario(
    name="backyard_garden_traverse",
    scene_usd="assets/eval_scenes/residential/backyard_garden.usd",
    robot_usd="assets/crab_hex.usd",
    category="residential",
    start_pos=[0.0, 0.0, 0.5],
    goal_pos=[12.0, 3.0, 0.5],
    difficulty="medium",
    max_time=45.0,
    max_falls=1,
    description="Navigate through backyard with garden beds and decorations"
)

# Add a stairs scenario
manager.add_scenario(
    name="front_porch_stairs_climb",
    scene_usd="assets/eval_scenes/stairs/front_porch.usd",
    robot_usd="assets/crab_hex.usd",
    category="stairs",
    start_pos=[0.0, 0.0, 0.5],
    goal_pos=[2.0, 0.0, 0.95],  # 3 stairs up
    difficulty="very_hard",
    min_progress=1.0,  # Even reaching first stair is progress
    max_time=60.0,
    max_falls=3,
    description="3-step front porch stairs, 15cm rise per step"
)

print("\n✓ Scenarios added successfully!")
print("Run: make eval-scenario SCENARIO=backyard_garden_traverse")
```

### AI-Friendly One-Liner

```python
# Add scenario in one line
from scenario_registry_manager import ScenarioRegistryManager
ScenarioRegistryManager().add_scenario(
    "my_scenario", 
    "assets/eval_scenes/residential/my_scene.usd",
    "assets/crab_hex.usd",
    "residential", [0, 0, 0.5], [10, 0, 0.5], 
    description="My test scenario"
)
```

---

## 7. AI-Runnable Commands

```bash
cd krabby-research

# Add a new scenario (Python method - RECOMMENDED)
python scripts/add_eval_scenario.py

# Or add scenario programmatically:
python -c "
from scripts.scenario_registry_manager import ScenarioRegistryManager
ScenarioRegistryManager().add_scenario(
    'my_new_scenario',
    'assets/eval_scenes/residential/my_scene.usd',
    'assets/crab_hex.usd',
    'residential',
    [0.0, 0.0, 0.5],
    [10.0, 0.0, 0.5],
    description='My new test scenario'
)
"

# Run full evaluation suite
make eval-full

# Evaluate single scenario
make eval-scenario SCENARIO=my_new_scenario

# Analyze failures
python scripts/analyze_failures.py \
    --results eval_results/*.json \
    --output failure_analysis.md
```
