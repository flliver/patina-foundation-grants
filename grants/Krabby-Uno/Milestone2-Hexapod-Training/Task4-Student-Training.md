# Task 4 — STUDENT MODEL TRAINING

Goal: Train student model with depth camera input. Achieve within 5% of Unitree performance on standard evaluation metrics.

Outputs
- Trained student model weights
- Training logs and metrics
- Reward function modifications (if any)
- Evaluation results comparing to Unitree baseline
- Analysis of performance gaps

Acceptance Criteria
- Student model trains successfully with depth input
- Performance within 5% of Unitree baseline on key metrics
- Model weights saved and loadable
- Comparison analysis documented
- **Early training sanity check**: Good progress early in training (reward trend, depth encoder active); document how to verify so time is not wasted on broken runs.

---

**NOTE**: All code examples in this document are AI-generated pseudocode with human comments where appropriate. Treat them as design guidance, not production-ready code. Actual implementation may require adjustments based on Isaac Sim API versions and specific requirements.

---

## 1. Student Model Overview

### Architecture Differences from Teacher
The student model uses:
- **Depth camera input**: 58×87 pixels (5046 dimensions)
- **Reduced proprioception**: No privileged information (terrain, friction, etc.)
- **Distillation loss**: Learns to mimic teacher's latent representations
- **Same action space**: 30 DOF joint positions

### Training Strategy
1. Pre-train teacher model (Task 3) ✓
2. Freeze teacher encoder
3. Train student encoder + policy to match teacher
4. Fine-tune end-to-end

---

## 2. Student Configuration

### Observation Space
```python
# Student observations (from Task 2)
@configclass
class CrabHexStudentObservationsCfg:
    """Student observation configuration."""
    
    @configclass
    class PolicyCfg(ObsGroup):
        """Core proprioceptive observations."""
        base_ang_vel = ObsTerm(func=mdp.base_ang_vel, scale=0.25)
        projected_gravity = ObsTerm(func=mdp.projected_gravity)
        joint_pos = ObsTerm(func=mdp.joint_pos_rel)
        joint_vel = ObsTerm(func=mdp.joint_vel_rel, scale=0.05)
        actions = ObsTerm(func=mdp.last_action)
        velocity_commands = ObsTerm(func=mdp.generated_commands)
        
        # NO height scan (replaced by depth camera)
        # NO privileged info (terrain type, friction, etc.)
    
    @configclass
    class DepthCameraPolicyCfg(ObsGroup):
        """Depth camera observations."""
        depth_features = ObsTerm(
            func=observations.image_features,
            params={
                "sensor_cfg": SceneEntityCfg("depth_camera"),
                "resize": (58, 87),
                "buffer_len": 2,
                "debug_vis": False,
            },
        )
    
    policy: PolicyCfg = PolicyCfg()
    depth_camera: DepthCameraPolicyCfg = DepthCameraPolicyCfg()
```

### Network Architecture
```python
# Student policy architecture
@configclass
class CrabHexStudentPPORunnerCfg(OnPolicyRunnerCfg):
    """Student PPO configuration with depth encoder."""
    
    num_steps_per_env = 24
    max_iterations = 15000  # Longer than teacher
    save_interval = 500
    experiment_name = "crab_hex_student"
    
    algorithm = PPOCfg(
        # Same as teacher
        value_loss_coef=1.0,
        clip_param=0.2,
        entropy_coef=0.01,
        num_learning_epochs=5,
        num_mini_batches=4,
        learning_rate=1e-3,
        gamma=0.99,
        lam=0.95,
        desired_kl=0.01,
        max_grad_norm=1.0,
    )
    
    policy = ActorCriticWithExtractorCfg(
        # Depth encoder
        depth_encoder_hidden_dims=[256, 128, 64],
        depth_encoder_activation="elu",
        
        # Proprioception encoder
        prop_encoder_hidden_dims=[128, 64],
        prop_encoder_activation="elu",
        
        # Fused policy
        actor_hidden_dims=[256, 128],
        critic_hidden_dims=[256, 128],
        activation="elu",
        
        # Distillation
        use_distillation=True,
        distillation_weight=0.5,
        teacher_checkpoint="logs/rsl_rl/crab_teacher_baseline/v1/model_10000.pt",
    )
```

---

## 3. Training Command

```bash
# Student training (full run)
python scripts/rsl_rl/train.py \
    --task Isaac-Crab-Hex-Student-v0 \
    --num_envs 4096 \
    --max_iterations 15000 \
    --seed 42 \
    --headless \
    --experiment_name crab_student_baseline \
    --run_name v1 \
    --teacher_checkpoint logs/rsl_rl/crab_teacher_baseline/v1/model_10000.pt

# Expected runtime: ~12-16 hours on RTX 5080
# Longer than teacher due to depth processing
```

### Training Phases

#### Early training sanity check
Ensure **good progress early in training**. Within the first 500–1500 iterations: depth encoder gradients should be non-zero; mean_reward should show an upward trend (not flat). Document how to verify (e.g. tensorboard, gradient norms) so time is not wasted on broken runs.

#### Phase 1: Depth Encoder Pre-training (0-3000 iterations)
**Goal**: Learn useful depth features

**Monitoring**:
```python
# Key metrics
depth_encoder_loss  # Should decrease from ~1.0 to ~0.1
reconstruction_error  # If using autoencoder pre-training
```

**Expected Behavior**:
- Robot may be unstable initially (depth features not useful yet)
- Mean reward may be lower than teacher early on
- Depth encoder gradients should be large

#### Phase 2: Policy Learning (3000-10000 iterations)
**Goal**: Learn locomotion policy using depth features

**Monitoring**:
```python
# Key metrics
mean_reward  # Should approach teacher performance
distillation_loss  # Should decrease
velocity_tracking_error  # Should match teacher
```

**Expected Behavior**:
- Performance gap to teacher narrows
- Depth features become more informative
- Policy becomes smoother

#### Phase 3: Fine-tuning (10000-15000 iterations)
**Goal**: Close remaining performance gap

**Monitoring**:
```python
# Key metrics
performance_gap  # Target: <5% vs teacher
fall_rate  # Should match teacher
success_rate_by_terrain  # Should be within 5% of teacher
```

---

## 4. Reward Function Adjustments

### Base Rewards (Same as Teacher)
```python
# Most rewards unchanged from teacher
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
    'leg_extension': -0.5,
    'body_height': -0.3,
    'undesired_contacts': -1.0,
    'foot_slip': -0.1,
}
```

### Student-Specific Adjustments

#### Distillation Loss
```python
# Add distillation reward (implicit in loss function)
'distillation': RewTerm(
    func=rewards.distillation_reward,
    weight=0.5,
    params={
        "teacher_checkpoint": "logs/rsl_rl/crab_teacher_baseline/v1/model_10000.pt",
    },
)
```

#### Depth-Aware Penalties
```python
# Penalize if depth camera view is blocked
'camera_occlusion': RewTerm(
    func=rewards.camera_occlusion_penalty,
    weight=-0.2,
    params={
        "sensor_cfg": SceneEntityCfg("depth_camera"),
        "min_visible_pixels": 0.7,  # 70% of pixels must be valid
    },
)

# Encourage head stability (camera mounted on body)
'head_stability': RewTerm(
    func=rewards.head_stability_reward,
    weight=0.1,
    params={
        "asset_cfg": SceneEntityCfg("robot"),
        "max_ang_vel": 2.0,  # rad/s
    },
)
```

### Custom Reward Functions
```python
# rewards.py
def camera_occlusion_penalty(
    env,
    sensor_cfg: SceneEntityCfg,
    min_visible_pixels: float,
) -> torch.Tensor:
    """Penalize if camera view is blocked."""
    camera = env.scene[sensor_cfg.name]
    
    # Get depth image
    depth = camera.data.output["distance_to_camera"]
    
    # Count valid pixels (not at max distance)
    valid_pixels = (depth < camera.cfg.max_distance * 0.95).float()
    visible_ratio = valid_pixels.mean(dim=(1, 2))
    
    # Penalize if below threshold
    penalty = torch.clamp(min_visible_pixels - visible_ratio, min=0.0)
    
    return -penalty

def head_stability_reward(
    env,
    asset_cfg: SceneEntityCfg,
    max_ang_vel: float,
) -> torch.Tensor:
    """Reward stable head/body orientation."""
    asset = env.scene[asset_cfg.name]
    
    # Get body angular velocity
    ang_vel = asset.data.root_ang_vel_w
    ang_vel_mag = torch.norm(ang_vel, dim=-1)
    
    # Reward if below threshold
    reward = torch.clamp(max_ang_vel - ang_vel_mag, min=0.0) / max_ang_vel
    
    return reward
```

---

## 5. Depth Processing

### Depth Encoder Architecture
```python
# depth_encoder.py
import torch
import torch.nn as nn

class DepthEncoder(nn.Module):
    """CNN encoder for depth images."""
    
    def __init__(
        self,
        input_shape=(58, 87),  # H, W
        hidden_dims=[256, 128, 64],
        output_dim=64,
        activation="elu",
    ):
        super().__init__()
        
        # Convolutional layers
        self.conv1 = nn.Conv2d(1, 32, kernel_size=5, stride=2, padding=2)
        self.conv2 = nn.Conv2d(32, 64, kernel_size=3, stride=2, padding=1)
        self.conv3 = nn.Conv2d(64, 128, kernel_size=3, stride=2, padding=1)
        
        # Calculate flattened size
        # After 3 stride-2 convs: (58/8, 87/8) = (7, 10)
        flat_size = 128 * 7 * 10
        
        # Fully connected layers
        self.fc1 = nn.Linear(flat_size, hidden_dims[0])
        self.fc2 = nn.Linear(hidden_dims[0], hidden_dims[1])
        self.fc3 = nn.Linear(hidden_dims[1], output_dim)
        
        self.activation = get_activation(activation)
    
    def forward(self, depth_image):
        """
        Args:
            depth_image: (batch, 58*87) flattened depth
        Returns:
            features: (batch, output_dim)
        """
        # Reshape to image
        x = depth_image.view(-1, 1, 58, 87)
        
        # Convolutional layers
        x = self.activation(self.conv1(x))
        x = self.activation(self.conv2(x))
        x = self.activation(self.conv3(x))
        
        # Flatten
        x = x.view(x.size(0), -1)
        
        # Fully connected
        x = self.activation(self.fc1(x))
        x = self.activation(self.fc2(x))
        x = self.fc3(x)
        
        return x
```

### Integration with Policy
```python
# actor_critic_with_extractor.py
class ActorCriticWithExtractor(nn.Module):
    """Policy with depth encoder."""
    
    def __init__(self, ...):
        super().__init__()
        
        # Depth encoder
        self.depth_encoder = DepthEncoder(
            input_shape=(58, 87),
            hidden_dims=[256, 128, 64],
            output_dim=64,
        )
        
        # Proprioception encoder
        self.prop_encoder = nn.Sequential(
            nn.Linear(prop_dim, 128),
            nn.ELU(),
            nn.Linear(128, 64),
            nn.ELU(),
        )
        
        # Fused policy (depth + prop features)
        self.actor = nn.Sequential(
            nn.Linear(64 + 64, 256),
            nn.ELU(),
            nn.Linear(256, 128),
            nn.ELU(),
            nn.Linear(128, action_dim),
        )
        
        self.critic = nn.Sequential(
            nn.Linear(64 + 64, 256),
            nn.ELU(),
            nn.Linear(256, 128),
            nn.ELU(),
            nn.Linear(128, 1),
        )
    
    def forward(self, obs):
        # Extract features
        depth_features = self.depth_encoder(obs['depth_camera'])
        prop_features = self.prop_encoder(obs['policy'])
        
        # Fuse
        fused = torch.cat([depth_features, prop_features], dim=-1)
        
        # Policy outputs
        actions = self.actor(fused)
        value = self.critic(fused)
        
        return actions, value
```

---

## 6. Evaluation Metrics

### Performance Targets (vs Unitree Baseline)
```python
metrics_targets = {
    # Primary metrics (must be within 5%)
    'mean_velocity': '±5% of Unitree',
    'fall_rate': '±5% of Unitree',
    'success_rate_flat': '±5% of Unitree',
    'success_rate_obstacles': '±5% of Unitree',
    'success_rate_gaps': '±5% of Unitree',
    
    # Secondary metrics (informational)
    'mean_torque': 'Expected 15-25% higher (more legs)',
    'action_smoothness': 'Similar to Unitree',
    'depth_encoder_utilization': '>80% (features used)',
}
```

### Comparison Script
```bash
# Compare student to Unitree baseline
python scripts/compare_student_to_baseline.py \
    --crab_student logs/rsl_rl/crab_student_baseline/v1/model_15000.pt \
    --unitree_student logs/rsl_rl/unitree_student_baseline/v1/model_15000.pt \
    --num_episodes 100 \
    --output student_comparison.md
```

---

## 7. Troubleshooting

### Common Issues

#### Depth Features Not Used
**Symptoms**: Depth encoder gradients near zero, performance same without depth

**Diagnosis**:
```python
# Check depth encoder activation
python scripts/debug_depth_encoder.py \
    --checkpoint logs/rsl_rl/crab_student_baseline/v1/model_5000.pt
```

**Solutions**:
- Increase distillation weight: `distillation_weight=1.0`
- Add depth reconstruction loss
- Reduce proprioception encoder capacity (force reliance on depth)

#### Performance Gap >5%
**Symptoms**: Student consistently worse than teacher/Unitree

**Solutions**:
```python
# Increase training iterations
max_iterations = 20000

# Increase depth encoder capacity
depth_encoder_hidden_dims = [512, 256, 128]

# Add curriculum for depth (start with easy scenes)
depth_curriculum = True
```

#### Unstable Training
**Symptoms**: Reward oscillates, high variance

**Solutions**:
```python
# Reduce learning rate
learning_rate = 5e-4

# Increase batch size
num_mini_batches = 8

# Add gradient clipping
max_grad_norm = 0.5
```

---

## 8. Success Criteria Checklist

- [ ] Student training completes 15,000 iterations
- [ ] Mean reward within 10% of teacher
- [ ] Velocity tracking within 5% of Unitree baseline
- [ ] Fall rate within 5% of Unitree baseline
- [ ] Success rate on flat terrain within 5% of Unitree
- [ ] Success rate on obstacles within 5% of Unitree
- [ ] Success rate on gaps within 5% of Unitree
- [ ] Depth encoder shows non-zero gradients
- [ ] Model weights saved and loadable
- [ ] Comparison analysis documented

---

## 9. AI-Runnable Commands

```bash
cd krabby-research/parkour

# Train student model
python scripts/rsl_rl/train.py \
    --task Isaac-Crab-Hex-Student-v0 \
    --num_envs 4096 \
    --max_iterations 15000 \
    --seed 42 \
    --headless \
    --experiment_name crab_student_baseline \
    --run_name v1 \
    --teacher_checkpoint logs/rsl_rl/crab_teacher_baseline/v1/model_10000.pt

# Monitor training
tensorboard --logdir logs/rsl_rl/crab_student_baseline

# Evaluate student
python scripts/rsl_rl/evaluation.py \
    --task Isaac-Crab-Hex-Student-v0 \
    --checkpoint logs/rsl_rl/crab_student_baseline/v1/model_15000.pt \
    --num_envs 100 \
    --record_video

# Compare to Unitree baseline
python scripts/compare_student_to_baseline.py \
    --crab logs/rsl_rl/crab_student_baseline/v1/model_15000.pt \
    --unitree logs/rsl_rl/unitree_student_baseline/v1/model_15000.pt \
    --output comparison.md
```

---

## 10. Expected Results

### Training Curves
- **Iterations 0-3000**: Depth encoder learning, reward 0 → 2
- **Iterations 3000-8000**: Policy learning, reward 2 → 5
- **Iterations 8000-12000**: Fine-tuning, reward 5 → 6
- **Iterations 12000-15000**: Convergence, reward 6-7

### Final Performance (vs Unitree)
- **Mean velocity**: 0.5-0.7 m/s (Unitree: 0.6-0.8 m/s) ✓ within 5%
- **Fall rate**: 6-8% (Unitree: 5-7%) ✓ within 5%
- **Flat success**: 92-95% (Unitree: 95-98%) ✓ within 5%
- **Obstacle success**: 80-85% (Unitree: 85-90%) ✓ within 5%
- **Gap success**: 65-70% (Unitree: 70-75%) ✓ within 5%
