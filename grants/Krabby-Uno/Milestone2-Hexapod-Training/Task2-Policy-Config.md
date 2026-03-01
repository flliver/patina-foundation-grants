# Task 2 — POLICY CONFIGURATION

Goal: Define PPO and policy configuration for the crab hexapod robot. Add RGBD/depth camera matching parkour robot. Demonstrate robot can initialize and move in training environment. Ensure configuration is correct for six legs, gait is documented, and depth camera aligns with the parkour HAL/depth pipeline (M3).

Outputs
- Policy configuration files (observation space, action space, rewards)
- Camera configuration and integration
- Initial training script showing robot movement
- Documentation of configuration choices (including gait and depth alignment)

Acceptance Criteria
- Policy config loads without errors
- RGBD/depth camera properly positioned, functional, and configurable (aligned with the parkour HAL/depth pipeline, M3)
- Robot spawns in training environment
- Training loop runs and robot exhibits movement
- Observation and action spaces properly sized for hexapod (6 legs)
- Gait configuration documented
- pytest validation test executes; robot "flops" in training env

---

**NOTE**: All code examples in this document are AI-generated pseudocode with human comments where appropriate. Treat them as design guidance, not production-ready code. Actual implementation may require adjustments based on Isaac Sim API versions and specific requirements.

---

## 1. Configuration Architecture

### File Structure
```
krabby-research/parkour/parkour_tasks/parkour_tasks/
├── crab_hexapod_task/
│   ├── __init__.py
│   ├── config/
│   │   ├── __init__.py
│   │   ├── crab_hex/
│   │   │   ├── __init__.py
│   │   │   ├── agents/
│   │   │   │   ├── rsl_rl_ppo_cfg.py  # PPO hyperparameters
│   │   │   │   ├── parkour_mdp_cfg.py     # MDP (obs, actions, rewards)
│   │   │   ├── crab_hex_env_cfg.py    # Environment config
│   │   │   └── crab_hex_scene_cfg.py  # Scene/robot config
```

### Relationship to Parkour Config
The crab config should:
- Inherit from / copy from parkour base classes where sensible
- Override hexapod-specific parameters (6 legs vs 4)
- Maintain compatibility with existing training infrastructure
- Use same camera/sensor setup for student model

### Configuration for six legs
All observation space, action space, and joint-name lists must be defined for a **six-legged** hexapod. Use 18 DOF or 30 DOF as per the actual design (e.g. 6 legs × 5 joints = 30). Ensure configs and code comments explicitly reference "6 legs" and the correct joint counts (e.g. 6 feet for contact, 30 joint positions). Do not reuse quadruped dimensions.

### Gait configuration
Document how **gait** is configured so it can be tuned for the hexapod:
- **Pattern**: e.g. alternating tripod (two tripods of three legs each), or other stable hex pattern.
- **Phase offsets**: phase relationship between leg groups (e.g. tripod A at 0°, tripod B at 180°).
- **Duty factor**: fraction of cycle each leg is in stance.
- **Where set**: command generator, reward weights, or curriculum config (document the exact config keys or code locations).

No new implementation mandate—ensure the existing reward/command setup supports tuning these for six legs.

### Depth camera alignment with parkour HAL (Milestone 3)
The **depth camera** (resolution, FOV, placement) must be **configurable** and aligned with the depth pipeline used in the parkour/HAL stack (Milestone 3). Depth camera settings should be reusable and match the HAL/depth pipeline used by the parkour runtime (e.g. ZED or equivalent depth observation layout) so the student model and future deployment use the same interface. Set exact resolution/FOV to match that pipeline.

### Reward functions and hexapod
Follow the existing Task 2 reward design. After running the Task 0 legged-gym (or Isaac Lab) hexapod example, **double-check** that reward terms (velocity tracking, stability, joint limits, etc.) work for a six-legged hexapod. Document any adjustments (e.g. scaling, extra penalty terms for 6 legs) in this task file.

---

## 2. Robot Scene Configuration

### Asset Definition
```python
# crab_hex_scene_cfg.py
from isaaclab.assets import ArticulationCfg
from isaaclab.sensors import CameraCfg, ContactSensorCfg, RayCasterCfg
from isaaclab.scene import InteractiveSceneCfg
from isaaclab.utils import configclass

@configclass
class CrabHexSceneCfg(InteractiveSceneCfg):
    """Configuration for crab hexapod scene (6 legs)."""
    
    # Robot articulation
    robot: ArticulationCfg = ArticulationCfg(
        prim_path="{ENV_REGEX_NS}/Robot",
        spawn=UsdFileCfg(
            usd_path="assets/crab_hex.usd",
            rigid_props=RigidBodyPropertiesCfg(
                disable_gravity=False,
                max_depenetration_velocity=1.0,
            ),
            articulation_props=ArticulationRootPropertiesCfg(
                enabled_self_collisions=True,
                solver_position_iteration_count=4,
                solver_velocity_iteration_count=0,
            ),
        ),
        init_state=ArticulationCfg.InitialStateCfg(
            pos=(0.0, 0.0, 0.5),  # Start 0.5m above ground
            joint_pos={
                ".*_hip_yaw": 0.0,
                ".*_hip_pitch": -0.785,
                ".*_knee": -1.57,
                ".*_knee_actuator_joint": 0.75,
                ".*_hip_actuator_joint": 0.20,
            },
            joint_vel={".*": 0.0},
        ),
        actuators={
            "hip_yaw": ImplicitActuatorCfg(
                joint_names_expr=[".*_hip_yaw"],
                effort_limit=40.0, velocity_limit=5.0, stiffness=25.0, damping=5.0,
            ),
            "hip_pitch": ImplicitActuatorCfg(
                joint_names_expr=[".*_hip_pitch"],
                effort_limit=40.0, velocity_limit=5.0, stiffness=25.0, damping=5.0,
            ),
            "knee": ImplicitActuatorCfg(
                joint_names_expr=[".*_knee"],
                effort_limit=40.0, velocity_limit=5.0, stiffness=25.0, damping=5.0,
            ),
            "prismatic": ImplicitActuatorCfg(
                joint_names_expr=[".*_actuator_joint"],
                effort_limit=200.0, velocity_limit=0.2, stiffness=1000.0, damping=100.0,
            ),
        },
    )
    
    contact_forces: ContactSensorCfg = ContactSensorCfg(
        prim_path="{ENV_REGEX_NS}/Robot/.*_tibia_tip",
        update_period=0.0, history_length=3, debug_vis=False,
    )
    
    height_scanner: RayCasterCfg = RayCasterCfg(
        prim_path="{ENV_REGEX_NS}/Robot/base_link",
        offset=RayCasterCfg.OffsetCfg(pos=(0.0, 0.0, 0.05)),
        attach_yaw_only=True,
        pattern_cfg=RayCasterCfg.GridPatternCfg(resolution=0.1, size=(1.6, 1.0)),
        max_distance=1.0, drift_range=(-0.0, 0.0), debug_vis=False,
    )
    
    # Depth camera: make resolution/FOV/placement configurable to match parkour HAL/depth pipeline (M3)
    depth_camera: CameraCfg = CameraCfg(
        prim_path="{ENV_REGEX_NS}/Robot/base_link/depth_camera",
        update_period=0.1,
        height=58, width=87,
        data_types=["distance_to_camera"],
        spawn=PinholeCameraCfg(
            focal_length=1.93, focus_distance=400.0,
            horizontal_aperture=6.0, clipping_range=(0.1, 10.0),
        ),
        offset=CameraCfg.OffsetCfg(
            pos=(0.25, 0.0, 0.05),
            rot=(1.0, 0.0, 0.0, 0.0),
            convention="ros",
        ),
    )
    
    ground = AssetBaseCfg(
        prim_path="/World/ground",
        spawn=GroundPlaneCfg(),
        init_state=AssetBaseCfg.InitialStateCfg(pos=(0.0, 0.0, 0.0)),
    )
```

---

## 3. MDP Configuration

### Observation Space
```python
# parkour_mdp_cfg.py – 6 legs: 30 joints, 6 feet contact
@configclass
class CrabHexObservationsCfg:
    @configclass
    class PolicyCfg(ObsGroup):
        base_ang_vel = ObsTerm(func=mdp.base_ang_vel, scale=0.25)
        base_lin_vel = ObsTerm(func=mdp.base_lin_vel, scale=2.0)
        projected_gravity = ObsTerm(func=mdp.projected_gravity)
        joint_pos = ObsTerm(func=mdp.joint_pos_rel, params={"asset_cfg": SceneEntityCfg("robot")})
        joint_vel = ObsTerm(func=mdp.joint_vel_rel, params={"asset_cfg": SceneEntityCfg("robot")}, scale=0.05)
        actions = ObsTerm(func=mdp.last_action)
        contact_forces = ObsTerm(func=mdp.contact_forces, params={"sensor_cfg": SceneEntityCfg("contact_forces"), "threshold": 2.0})
        height_scan = ObsTerm(func=mdp.height_scan, params={"sensor_cfg": SceneEntityCfg("height_scanner")}, clip=(-1.0, 1.0))
        velocity_commands = ObsTerm(func=mdp.generated_commands, params={"command_name": "base_velocity"})
    policy: PolicyCfg = PolicyCfg()
    
    @configclass
    class DepthCameraPolicyCfg(ObsGroup):
        depth_features = ObsTerm(
            func=observations.image_features,
            params={"sensor_cfg": SceneEntityCfg("depth_camera"), "resize": (58, 87), "buffer_len": 2, "debug_vis": False},
        )
    depth_camera: DepthCameraPolicyCfg = DepthCameraPolicyCfg()
```

### Action Space
```python
@configclass
class CrabHexActionsCfg:
    joint_pos = mdp.JointPositionActionCfg(
        asset_name="robot",
        joint_names=[".*"],
        scale=0.25,
        use_default_offset=True,
    )
```

### Observation Space Dimensions
- Teacher: base_ang_vel 3 + base_lin_vel 3 + projected_gravity 3 + joint_pos 30 + joint_vel 30 + actions 30 + contact_forces 6 + height_scan 132 + velocity_commands 3 → **240 dimensions**
- Student: same proprio + depth 58×87 = **5046** depth pixels
- Action: **30 DOF** (6 legs × 5 joints)

---

## 4. Reward Configuration

### Reward Terms
```python
@configclass
class CrabHexRewardsCfg:
    track_lin_vel_xy = RewTerm(func=mdp.track_lin_vel_xy_exp, weight=1.5, params={"command_name": "base_velocity", "std": 0.5})
    track_ang_vel_z = RewTerm(func=mdp.track_ang_vel_z_exp, weight=0.75, params={"command_name": "base_velocity", "std": 0.5})
    lin_vel_z = RewTerm(func=mdp.lin_vel_z_l2, weight=-2.0)
    ang_vel_xy = RewTerm(func=mdp.ang_vel_xy_l2, weight=-0.05)
    flat_orientation = RewTerm(func=mdp.flat_orientation_l2, weight=-1.0)
    joint_torques = RewTerm(func=mdp.joint_torques_l2, weight=-0.00001)
    joint_vel = RewTerm(func=mdp.joint_vel_l2, weight=-0.0001)
    joint_acc = RewTerm(func=mdp.joint_acc_l2, weight=-2.5e-7)
    action_rate = RewTerm(func=mdp.action_rate_l2, weight=-0.01)
    # Hexapod-specific
    leg_extension = RewTerm(func=rewards.leg_extension_penalty, weight=-0.5, params={"asset_cfg": SceneEntityCfg("robot"), "max_extension": 1.0})
    body_height = RewTerm(func=rewards.body_height_penalty, weight=-0.3, params={"asset_cfg": SceneEntityCfg("robot"), "target_height": 0.3, "tolerance": 0.1})
    undesired_contacts = RewTerm(func=mdp.undesired_contacts, weight=-1.0, params={"sensor_cfg": SceneEntityCfg("contact_forces", body_names=["base", ".*_femur", ".*_tibia"]), "threshold": 1.0})
    foot_slip = RewTerm(func=mdp.foot_slip, weight=-0.1, params={"sensor_cfg": SceneEntityCfg("contact_forces"), "asset_cfg": SceneEntityCfg("robot", body_names=".*_tibia_tip")})
```

### Custom Reward Functions (hexapod)
Implement in a `rewards` module (e.g. `rewards.py`). Use 6 feet: `['lf', 'lm', 'lr', 'rf', 'rm', 'rr']` and body names `.*_tibia_tip`. After Task 0 hexapod example, verify scaling/penalties for six legs and document any changes.

```python
# rewards.py (hexapod-specific)
import torch
from isaaclab.managers import SceneEntityCfg

def leg_extension_penalty(env, asset_cfg: SceneEntityCfg, max_extension: float) -> torch.Tensor:
    """Penalize legs extending too far from body center (XY plane)."""
    asset = env.scene[asset_cfg.name]
    foot_bodies = [f"{leg}_tibia_tip" for leg in ['lf', 'lm', 'lr', 'rf', 'rm', 'rr']]
    foot_positions = asset.data.body_pos_w[:, asset.find_bodies(foot_bodies)[0]]
    body_pos = asset.data.root_pos_w
    distances = torch.norm(foot_positions[:, :, :2] - body_pos[:, None, :2], dim=-1)
    penalty = torch.sum(torch.clamp(distances - max_extension, min=0.0), dim=-1)
    return -penalty

def body_height_penalty(env, asset_cfg: SceneEntityCfg, target_height: float, tolerance: float) -> torch.Tensor:
    """Penalize body height deviating from target."""
    asset = env.scene[asset_cfg.name]
    body_height = asset.data.root_pos_w[:, 2]
    deviation = torch.abs(body_height - target_height)
    penalty = torch.clamp(deviation - tolerance, min=0.0)
    return -penalty
```

---

## 5. PPO Configuration

```python
# rsl_rl_ppo_cfg.py
@configclass
class CrabHexPPORunnerCfg(OnPolicyRunnerCfg):
    num_steps_per_env = 24
    max_iterations = 10000
    save_interval = 500
    experiment_name = "crab_hex"
    run_name = ""
    logger = "tensorboard"
    algorithm = PPOCfg(
        value_loss_coef=1.0, use_clipped_value_loss=True, clip_param=0.2,
        entropy_coef=0.01, num_learning_epochs=5, num_mini_batches=4,
        learning_rate=1e-3, schedule="adaptive", gamma=0.99, lam=0.95,
        desired_kl=0.01, max_grad_norm=1.0,
    )
    policy = ActorCriticCfg(
        init_noise_std=1.0,
        actor_hidden_dims=[512, 256, 128],
        critic_hidden_dims=[512, 256, 128],
        activation="elu",
    )
```

---

## 6. Environment Configuration

```python
# crab_hex_env_cfg.py
@configclass
class CrabHexEnvCfg(ParkourManagerBasedRLEnvCfg):
    scene: CrabHexSceneCfg = CrabHexSceneCfg(num_envs=4096, env_spacing=4.0)
    observations: CrabHexObservationsCfg = CrabHexObservationsCfg()
    actions: CrabHexActionsCfg = CrabHexActionsCfg()
    rewards: CrabHexRewardsCfg = CrabHexRewardsCfg()
    episode_length_s = 20.0
    decimation = 4
    sim = SimulationCfg(
        dt=1/200,
        render_interval=decimation,
        physics_material=RigidBodyMaterialCfg(static_friction=1.0, dynamic_friction=1.0, restitution=0.0),
    )
```

---

## 7. Registration

```python
# __init__.py
import gymnasium as gym
from . import agents
from .crab_hex_env_cfg import CrabHexEnvCfg

gym.register(
    id="Isaac-Crab-Hex-Teacher-v0",
    entry_point="parkour_isaaclab.envs:ParkourManagerBasedRLEnv",
    disable_env_checker=True,
    kwargs={"env_cfg_entry_point": CrabHexEnvCfg, "rsl_rl_cfg_entry_point": agents.rsl_rl_ppo_cfg.CrabHexPPORunnerCfg},
)
gym.register(
    id="Isaac-Crab-Hex-Student-v0",
    entry_point="parkour_isaaclab.envs:ParkourManagerBasedRLEnv",
    disable_env_checker=True,
    kwargs={"env_cfg_entry_point": CrabHexEnvCfg, "rsl_rl_cfg_entry_point": agents.rsl_rl_ppo_cfg.CrabHexStudentPPORunnerCfg},
)
```

---

## 8. Smoke Test Script

```python
# test_crab_config.py
import pytest
import gymnasium as gym
import torch

def test_teacher_env_runs():
    """Teacher env loads, runs, produces rewards; robot moves (not frozen)."""
    env = gym.make("Isaac-Crab-Hex-Teacher-v0", num_envs=4, headless=True)
    obs, _ = env.reset()
    assert 'policy' in obs
    assert obs['policy'].shape == (4, 240)
    initial_obs = obs['policy'].clone()
    for _ in range(100):
        actions = torch.randn((4, 30)) * 0.1
        obs, rewards, dones, truncated, info = env.step(actions)
    obs_change = torch.norm(obs['policy'] - initial_obs, dim=1).mean()
    assert obs_change > 0.1
    assert rewards.abs().sum().item() > 0.0
    env.close()

def test_student_depth_camera():
    """Student env has depth camera at 58×87."""
    env = gym.make("Isaac-Crab-Hex-Student-v0", num_envs=2, headless=True)
    obs, _ = env.reset()
    assert 'depth_camera' in obs
    assert obs['depth_camera'].shape == (2, 5046)
    env.close()

def test_parallel_scaling():
    """Scales to many parallel envs."""
    env = gym.make("Isaac-Crab-Hex-Teacher-v0", num_envs=64, headless=True)
    obs, _ = env.reset()
    assert obs['policy'].shape[0] == 64
    env.close()
```

### Running Tests
```bash
pytest test_crab_config.py -v
# Or: pytest test_crab_config.py::test_teacher_env_runs -v -s
```

---

## 9. Success Criteria Checklist

- [ ] All config files created and importable; robot USD loads
- [ ] Observation space 240 dims (teacher), 5046 depth (student); action 30 DOF (6 legs × 5 joints)
- [ ] Depth camera configurable and aligned with parkour HAL/depth pipeline (Milestone 3)
- [ ] Gait pattern and config location documented
- [ ] Reward terms double-checked for hexapod after Task 0; adjustments documented
- [ ] All pytest smoke tests pass; robot exhibits movement in training env

---

## 10. AI-Runnable Commands

```bash
cd krabby-research/parkour
mkdir -p parkour_tasks/parkour_tasks/crab_hexapod_task/config/crab_hex/agents
pytest test_crab_config.py -v
python scripts/rsl_rl/train.py --task Isaac-Crab-Hex-Teacher-v0 --num_envs 64 --max_iterations 10 --headless
```
