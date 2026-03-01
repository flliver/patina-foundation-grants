# Task 3 — TEACHER MODEL TRAINING

Goal: Train teacher model with crab hexapod through same curriculum as Unitree baseline. Tune reward function for stable, compact locomotion.

Outputs
- Trained teacher model weights
- Training logs and metrics
- Reward function modifications (if any)
- Evaluation results on curriculum tasks
- Comparison metrics vs. Unitree baseline

Acceptance Criteria
- Teacher model completes training curriculum
- Robot successfully navigates gap jumps
- Reward function produces stable, compact locomotion
- Performance metrics documented
- Model weights saved and loadable
- **Early training sanity check**: Good progress early in training (e.g. reward trend upward, no immediate collapse); document how to verify (tensorboard, first N iterations) so time is not wasted on broken runs.

---

**NOTE**: All code examples in this document are AI-generated pseudocode with human comments where appropriate. Treat them as design guidance, not production-ready code. Actual implementation may require adjustments based on Isaac Sim API versions and specific requirements.

---

## 1. Training Setup

### Curriculum Overview
The parkour curriculum progresses through:
1. Flat terrain (baseline walking)
2. Low obstacles (stepping over)
3. Uneven terrain (height variation)
4. Gap jumps (stepping across voids)
5. Mixed parkour (combination)

### Expected Adaptations for Hexapod
- **Stability advantage**: 6 legs provide more stable base than quadruped
- **Gap crossing**: Will likely step across rather than jump (longer reach)
- **Speed trade-off**: May be slower but more stable than Unitree
- **Compact stance**: Need to encourage legs stay close to body

---

## 2. Baseline Training Command

```bash
# Teacher training (full run)
python scripts/rsl_rl/train.py \
    --task Isaac-Crab-Hex-Teacher-v0 \
    --num_envs 4096 \
    --max_iterations 10000 \
    --seed 42 \
    --headless \
    --experiment_name crab_teacher_baseline \
    --run_name v1

# Expected runtime: ~8-12 hours on RTX 5080
# Checkpoint saved every 500 iterations
```

### Training Monitoring
```bash
# Launch tensorboard
tensorboard --logdir logs/rsl_rl/crab_teacher_baseline

# Key metrics to watch:
# - mean_reward: should increase from ~0 to 5-10
# - episode_length: should reach max (500 steps)
# - lin_vel_tracking: should approach 1.0
# - fall_rate: should decrease to <5%
```

### Early training sanity check
Ensure **good progress early in training** so time is not wasted on broken runs. Within the first 500–1000 iterations: mean_reward should show an upward trend (not flat or collapsing); fall_rate should not be stuck at 100%. Document how to check (e.g. tensorboard over first N iterations, or a short script that runs 500 steps and asserts reward > threshold). If reward stays near zero or fall rate is extreme, stop and fix config/robot/rewards before continuing.

---

## 3. Reward Function Tuning

### Initial Reward Weights (from Task 2)
```python
rewards = {
    'track_lin_vel_xy': 1.5,
    'track_ang_vel_z': 0.75,
    'lin_vel_z': -2.0,
    'ang_vel_xy': -0.05,
    'flat_orientation': -1.0,
    'joint_torques': -0.00001,
    'joint_vel': -0.0001,
    'joint_acc': -2.5e-7,
    'action_rate': -0.01,
    'leg_extension': -0.5,  # NEW: hexapod-specific
    'body_height': -0.3,    # NEW: hexapod-specific
    'undesired_contacts': -1.0,
    'foot_slip': -0.1,
}
```

### Tuning Process

#### Phase 1: Baseline Gait (Iterations 0-2000)
**Goal**: Get robot walking on flat terrain

**Expected Issues**:
- Legs may splay out too far
- Body may be too high (unstable)
- Gait may be uncoordinated

**Tuning Adjustments**:
```python
# If legs extend too far:
'leg_extension': -1.0,  # Increase from -0.5

# If body too high:
'body_height': -0.6,  # Increase from -0.3

# If gait uncoordinated:
'action_rate': -0.05,  # Increase from -0.01 (smoother actions)
```

#### Phase 2: Obstacle Navigation (Iterations 2000-5000)
**Goal**: Navigate low obstacles and uneven terrain

**Expected Issues**:
- Robot may try to go around obstacles instead of over
- Foot placement may be imprecise
- May lose balance on uneven ground

**Tuning Adjustments**:
```python
# Encourage forward progress
'track_lin_vel_xy': 2.0,  # Increase from 1.5

# Penalize foot slip more
'foot_slip': -0.3,  # Increase from -0.1

# Add foot clearance reward
'foot_clearance': RewTerm(
    func=rewards.foot_clearance_reward,
    weight=0.5,
    params={
        "sensor_cfg": SceneEntityCfg("contact_forces"),
        "asset_cfg": SceneEntityCfg("robot", body_names=".*_tibia_tip"),
        "target_height": 0.1,  # 10cm clearance during swing
    },
)
```

#### Phase 3: Gap Jumps (Iterations 5000-8000)
**Goal**: Successfully cross gaps

**Expected Behavior**:
- Hexapod will likely STEP across gaps (not jump)
- Needs to extend legs forward to reach far side
- Must maintain balance during transition

**Tuning Adjustments**:
```python
# Temporarily relax leg extension penalty during gaps
# (This requires detecting gap terrain - see curriculum section)

# Add gap-specific reward
'gap_crossing': RewTerm(
    func=rewards.gap_crossing_reward,
    weight=2.0,
    params={
        "parkour_name": "base_parkour",
        "min_progress": 0.5,  # Must make forward progress
    },
)

# Reduce body height penalty (may need to reach)
'body_height': -0.2,  # Decrease from -0.6
```

---

## 4. Custom Reward Functions

### Foot Clearance Reward
```python
# rewards.py
def foot_clearance_reward(
    env,
    sensor_cfg: SceneEntityCfg,
    asset_cfg: SceneEntityCfg,
    target_height: float,
) -> torch.Tensor:
    """Reward feet being lifted during swing phase."""
    asset = env.scene[asset_cfg.name]
    contact_sensor = env.scene.sensors[sensor_cfg.name]
    
    # Get foot heights
    foot_bodies = [f"{leg}_tibia_tip" for leg in ['lf', 'lm', 'lr', 'rf', 'rm', 'rr']]
    foot_positions = asset.data.body_pos_w[:, asset.find_bodies(foot_bodies)[0]]
    foot_heights = foot_positions[:, :, 2]
    
    # Get contact states
    contact_forces = contact_sensor.data.net_forces_w[:, sensor_cfg.body_ids]
    in_contact = torch.norm(contact_forces, dim=-1) > 1.0
    
    # Reward clearance for feet not in contact
    clearance = torch.clamp(foot_heights - target_height, min=0.0, max=0.2)
    reward = torch.where(in_contact, torch.zeros_like(clearance), clearance)
    
    return torch.sum(reward, dim=-1)

def gap_crossing_reward(
    env,
    parkour_name: str,
    min_progress: float,
) -> torch.Tensor:
    """Reward making progress across gaps."""
    parkour_event = env.parkour_manager.get_term(parkour_name)
    
    # Check if currently on gap terrain
    terrain_names = parkour_event.env_per_terrain_name
    on_gap = (terrain_names == 'parkour_gap')
    
    # Get forward velocity
    asset = env.scene['robot']
    forward_vel = asset.data.root_lin_vel_b[:, 0]
    
    # Reward forward progress only on gaps
    reward = torch.where(
        on_gap,
        torch.clamp(forward_vel - min_progress, min=0.0),
        torch.zeros_like(forward_vel)
    )
    
    return reward
```

---

## 5. Curriculum Configuration

### Terrain Progression
```python
# Update parkour_mdp_cfg.py
@configclass
class CrabHexParkourEventsCfg:
    """Parkour curriculum for crab hexapod."""
    
    base_parkour = parkours.ParkourEventsCfg(
        asset_name='robot',
        curriculum_stages=[
            # Stage 1: Flat (0-2000 iterations)
            {
                'name': 'flat',
                'weight': 1.0,
                'max_iteration': 2000,
            },
            # Stage 2: Low obstacles (2000-4000)
            {
                'name': 'obstacles_low',
                'weight': 0.5,
                'max_iteration': 4000,
                'params': {
                    'obstacle_height_range': (0.05, 0.15),
                },
            },
            # Stage 3: Uneven terrain (4000-6000)
            {
                'name': 'uneven',
                'weight': 0.7,
                'max_iteration': 6000,
                'params': {
                    'height_variation': 0.2,
                },
            },
            # Stage 4: Gaps (6000-8000)
            {
                'name': 'gaps',
                'weight': 0.3,
                'max_iteration': 8000,
                'params': {
                    'gap_width_range': (0.3, 0.6),  # Hexapod can step across
                },
            },
            # Stage 5: Mixed (8000+)
            {
                'name': 'mixed',
                'weight': 1.0,
                'max_iteration': 10000,
            },
        ],
    )
```

---

## 6. Evaluation Metrics

### Success Criteria
```python
# Metrics to track during training
metrics = {
    # Locomotion
    'mean_velocity': 'target: 0.5-0.8 m/s',
    'velocity_tracking_error': 'target: <0.2 m/s',
    'angular_velocity_tracking': 'target: <0.3 rad/s error',
    
    # Stability
    'fall_rate': 'target: <5%',
    'body_height_std': 'target: <0.05m',
    'orientation_error': 'target: <10 degrees',
    
    # Efficiency
    'mean_torque': 'compare to Unitree baseline',
    'action_smoothness': 'target: low action_rate penalty',
    
    # Hexapod-specific
    'leg_extension_mean': 'target: <0.8m from body',
    'body_height_mean': 'target: 0.25-0.35m',
    
    # Curriculum
    'flat_success_rate': 'target: >95%',
    'obstacle_success_rate': 'target: >85%',
    'uneven_success_rate': 'target: >80%',
    'gap_success_rate': 'target: >70%',
}
```

### Comparison to Unitree Baseline
```bash
# Run Unitree baseline for comparison
python scripts/rsl_rl/train.py \
    --task Isaac-Extreme-Parkour-Teacher-Unitree-Go2-v0 \
    --num_envs 4096 \
    --max_iterations 10000 \
    --seed 42 \
    --headless \
    --experiment_name unitree_baseline \
    --run_name v1

# Compare metrics
python scripts/compare_training_runs.py \
    --run1 logs/rsl_rl/crab_teacher_baseline/v1 \
    --run2 logs/rsl_rl/unitree_baseline/v1 \
    --output comparison_report.md
```

---

## 7. Troubleshooting

### Common Issues

#### Robot Falls Immediately
**Symptoms**: High fall rate (>50%) in first 1000 iterations

**Diagnosis**:
```python
# Check initial joint positions
python scripts/debug_initial_state.py --task Isaac-Crab-Hex-Teacher-v0
```

**Solutions**:
- Adjust init_state joint positions (may need legs more bent)
- Increase actuator stiffness/damping
- Add initial stability reward bonus

#### Legs Splay Out
**Symptoms**: leg_extension penalty dominates, robot unstable

**Solutions**:
```python
# Increase leg extension penalty
'leg_extension': -2.0,  # From -0.5

# Add joint limit penalty
'joint_limits': RewTerm(
    func=mdp.joint_pos_limits,
    weight=-1.0,
)
```

#### Slow Convergence
**Symptoms**: Mean reward plateaus below 3.0

**Solutions**:
- Increase learning rate: `learning_rate=3e-3`
- Reduce num_mini_batches: `num_mini_batches=2`
- Increase tracking reward: `track_lin_vel_xy=3.0`

#### Can't Cross Gaps
**Symptoms**: Success rate on gaps <30%

**Solutions**:
- Reduce gap width range: `gap_width_range=(0.2, 0.4)`
- Add gap_crossing reward (see Section 4)
- Temporarily disable leg_extension penalty on gap terrain

---

## 8. Training Checkpoints

### Checkpoint Schedule
```python
# Save checkpoints every 100 iterations
save_interval = 100

# Key checkpoints to preserve:
checkpoints = {
    'iter_2000': 'Flat terrain mastery',
    'iter_4000': 'Obstacle navigation',
    'iter_6000': 'Uneven terrain',
    'iter_8000': 'Gap crossing',
    'iter_10000': 'Final model',
}
```

### Checkpoint Evaluation
```bash
# Evaluate specific checkpoint
python scripts/rsl_rl/evaluation.py \
    --task Isaac-Crab-Hex-Teacher-v0 \
    --checkpoint logs/rsl_rl/crab_teacher_baseline/v1/model_2000.pt \
    --num_envs 100 \
    --record_video

# Generate metrics report
python scripts/evaluate_checkpoint.py \
    --checkpoint logs/rsl_rl/crab_teacher_baseline/v1/model_2000.pt \
    --output checkpoint_2000_report.md
```

---

## 9. Success Criteria Checklist

- [ ] Training completes 10,000 iterations without crashes
- [ ] Mean reward reaches >5.0 by iteration 8000
- [ ] Fall rate <5% on flat terrain
- [ ] Fall rate <15% on mixed parkour
- [ ] Successfully crosses gaps >70% of the time
- [ ] Velocity tracking error <0.2 m/s
- [ ] Leg extension stays <0.8m from body
- [ ] Body height stable at 0.25-0.35m
- [ ] Model weights saved and loadable
- [ ] Comparison to Unitree shows <10% performance gap

---

## 10. AI-Runnable Commands

```bash
cd krabby-research/parkour

# Full training run
python scripts/rsl_rl/train.py \
    --task Isaac-Crab-Hex-Teacher-v0 \
    --num_envs 4096 \
    --max_iterations 10000 \
    --seed 42 \
    --headless \
    --experiment_name crab_teacher_baseline \
    --run_name v1

# Monitor training
tensorboard --logdir logs/rsl_rl/crab_teacher_baseline

# Evaluate final model
python scripts/rsl_rl/evaluation.py \
    --task Isaac-Crab-Hex-Teacher-v0 \
    --checkpoint logs/rsl_rl/crab_teacher_baseline/v1/model_10000.pt \
    --num_envs 100 \
    --record_video \
    --output evaluation_report.md

# Compare to Unitree
python scripts/compare_models.py \
    --crab logs/rsl_rl/crab_teacher_baseline/v1/model_10000.pt \
    --unitree logs/rsl_rl/unitree_baseline/v1/model_10000.pt \
    --output comparison.md
```

---

## 11. Expected Results

### Training Curves
- **Iterations 0-2000**: Rapid improvement, mean reward 0 → 3
- **Iterations 2000-5000**: Steady improvement, mean reward 3 → 6
- **Iterations 5000-8000**: Slower improvement, mean reward 6 → 7
- **Iterations 8000-10000**: Plateau, mean reward 7-8

### Final Performance
- **Flat terrain**: 95%+ success, 0.6-0.8 m/s velocity
- **Obstacles**: 85%+ success, 0.4-0.6 m/s velocity
- **Uneven**: 80%+ success, 0.3-0.5 m/s velocity
- **Gaps**: 70%+ success, 0.2-0.4 m/s velocity

### Comparison to Unitree
- **Speed**: Crab 10-20% slower (expected due to more legs)
- **Stability**: Crab 5-10% more stable (lower fall rate)
- **Energy**: Crab 15-25% higher torque (more actuators)
- **Gap crossing**: Crab steps across, Unitree jumps (different strategy)
