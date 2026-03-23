# Patina Foundation Grant - Krabby-Uno Milestone 11: Scene Reconstruction & Locomotion Benchmarking

## Grant Overview
Build a simulation pipeline that converts monocular phone video into physically accurate IsaacSim environments and evaluates locomotion models on them. The robot uses depth + PPO for locomotion, so the priority is collision geometry fidelity, not visual rendering quality. The pipeline uses COLMAP for structure-from-motion and dense MVS reconstruction, generates conditioned collision meshes, exports to USD, and imports into IsaacSim. Two locomotion models (Extreme Parkour and Holosoma quadruped) are then evaluated on identical reconstructed environments using a hexapod embodiment, producing comparable metrics and demo videos. Holosoma runs without vision (proprioception-only), providing a baseline comparison against Extreme Parkour's depth-based approach. NVIDIA NuRec (COLMAP + 3DGUT) provides the reference workflow.

## Why is this Important?
- Bridges the real-to-sim gap by turning ordinary phone video into physically walkable simulation environments, making environment creation accessible to anyone with a camera.
- Provides an apples-to-apples comparison framework for locomotion models on identical environments, enabling rigorous benchmarking.
- Extends state-of-the-art quadruped locomotion models to hexapod embodiment, broadening the research landscape for multi-legged robots.
- Focuses on collision geometry quality over visual fidelity — the robot perceives via depth, so what matters is that the physics mesh accurately represents the real-world surfaces.

## Tasks
### Task 0 - Scene Capture & COLMAP Sparse Reconstruction (Photos → Sparse Point Cloud)
#### Narrative
Capture photos of 2–3 indoor/outdoor spaces using a phone camera. Run COLMAP's Structure-from-Motion (SfM) pipeline to extract camera poses and a sparse point cloud. COLMAP is the industry-standard SfM tool used by NVIDIA's NuRec workflow, GaussGym, and most real-to-sim pipelines. It produces reliable camera intrinsics/extrinsics that are required for dense reconstruction. Use pinhole or simple pinhole camera model for compatibility with downstream tools.
#### Tooling
- COLMAP (`colmap`): SfM pipeline. Key commands:
  - `colmap feature_extractor` — SIFT feature detection with `--ImageReader.camera_model PINHOLE`
  - `colmap exhaustive_matcher` — feature matching (GPU-accelerated with `--SiftMatching.use_gpu 1`)
  - `colmap mapper` — sparse reconstruction, outputs camera poses + sparse point cloud to `sparse/` directory
  - `colmap gui` — optional visual verification of sparse reconstruction
- Input: JPEG/PNG photos (convert HEIC if iPhone: Settings → Camera → Formats → Most Compatible)
- Output: `sparse/` directory with camera poses, intrinsics, and sparse point cloud
#### Acceptance Criteria
- Photos captured following photogrammetry best practices: slow steady motion, ~60% overlap between shots, multiple heights/angles, locked focus/exposure, avoiding reflective surfaces / motion blur / low-texture regions.
- COLMAP sparse reconstruction completes successfully; camera poses estimated for all (or nearly all) input images.
- Sparse point cloud visually verified in COLMAP GUI or MeshLab; scene structure is recognizable across 2–3 scenes.

### Task 1 - Dense Reconstruction & Collision Mesh Generation (Sparse → Dense → Mesh)
#### Narrative
Run COLMAP's Multi-View Stereo (MVS) pipeline to produce a dense point cloud from the sparse reconstruction, then generate a collision-quality triangle mesh. Since the robot uses depth + PPO, the mesh must be physically accurate (walkable surfaces, correct geometry) rather than visually pretty. COLMAP provides built-in dense reconstruction (`image_undistorter` → `patch_match_stereo` → `stereo_fusion`) and meshing (`poisson_mesher` or `delaunay_mesher`). For better mesh quality, export the dense point cloud and run Poisson reconstruction in Open3D with more control over parameters. The two COLMAP meshers can also be combined: Delaunay first to filter outliers, then Poisson for a smooth surface.
#### Tooling
- COLMAP dense reconstruction:
  - `colmap image_undistorter` — undistort images using estimated camera parameters
  - `colmap patch_match_stereo` — compute dense depth maps via multi-view stereo
  - `colmap stereo_fusion` — fuse depth maps into a dense point cloud (`fused.ply`)
- COLMAP meshing (option A — quick):
  - `colmap delaunay_mesher --input_type dense` — Delaunay triangulation, good for outlier filtering
  - `colmap poisson_mesher` — Poisson surface reconstruction, produces smooth watertight mesh
- Open3D meshing (option B — more control):
  - `open3d.geometry.PointCloud.estimate_normals()` — compute normals if missing from fusion output
  - `open3d.geometry.TriangleMesh.create_from_point_cloud_poisson(pcd, depth=9)` — Poisson reconstruction with tunable resolution (depth 8–10 for room-scale)
  - Density-based cropping to remove Poisson extrapolation artifacts at scene boundaries
- Output: triangle mesh as OBJ or PLY
#### Acceptance Criteria
- COLMAP MVS produces a dense point cloud (`fused.ply`) for each scene; point density is sufficient to capture floor, walls, and obstacles.
- Mesh reconstruction (COLMAP or Open3D Poisson) produces a watertight triangle mesh from each dense point cloud.
- Meshes are visually inspected in MeshLab/Blender; walkable surfaces (floors, ramps, stairs) are geometrically accurate.

### Task 2 - Mesh Conditioning & USD Export (Mesh → Collision-Ready USD)
#### Narrative
Raw reconstructed meshes contain noise, floaters, holes, and non-walkable topology. Apply conditioning focused on physics accuracy: remove outliers/floaters, fill holes, smooth surfaces, decimate to manageable triangle counts, and generate collision proxies. Since the robot uses depth sensing, the mesh IS the environment — there is no separate visual layer. Convert conditioned meshes to USD with correct scale (monocular reconstruction has scale ambiguity — must be manually calibrated), Z-up orientation, and origin alignment. Add physics properties (rigid body, collision mesh) and import into IsaacSim.
#### Tooling
- Open3D:
  - `remove_degenerate_triangles()`, `remove_unreferenced_vertices()` — basic cleanup
  - `filter_smooth_taubin()` — surface smoothing without shrinkage (preferred over Laplacian)
  - `simplify_quadric_decimation(target)` — reduce triangle count while preserving geometry
- PyMeshLab (`pymeshlab`):
  - `remove_isolated_pieces_wrt_diameter()` — floater/outlier removal
  - `close_holes(maxholesize=N)` — hole filling
  - `simplification_quadric_edge_collapse_decimation()` — decimation with quality bounds
- trimesh (`trimesh`):
  - `trimesh.repair.fill_holes()`, `trimesh.repair.fix_normals()` — quick repair
  - `trimesh.convex.convex_hull()` — simple collision proxy
  - `trimesh.decomposition.convex_decomposition()` — V-HACD approximate convex decomposition for complex collision shapes
- USD conversion:
  - Isaac Lab `MeshConverter` (`isaaclab.sim.converters.mesh_converter`) — converts OBJ/STL/FBX → USD with physics properties and collision meshes in instanceable format. Preferred path.
  - `pxr` (OpenUSD Python API) — manual control over `UsdGeom`, `UsdPhysics`, `PhysxSchema` if `MeshConverter` doesn't handle collision modes correctly
- Scale calibration: place a known-size reference object in the scene during capture, or measure a known distance post-reconstruction and apply a uniform scale factor
#### Acceptance Criteria
- Conditioning pipeline removes floaters, fills holes, smooths surfaces, and decimates; before/after comparison documented for each scene.
- Collision mesh generated for each environment; convex decomposition applied where needed for non-convex geometry.
- USD export has correct scale (validated against known reference), Z-up orientation, physics properties (rigid body + mesh collider), and loads in IsaacSim without errors.
- Robot can spawn on the mesh floor and depth sensor returns plausible readings from the environment geometry.

### Task 3 - Locomotion Model Integration (Dockerized)
#### Narrative
Integrate Extreme Parkour and Holosoma quadruped as the two evaluation models. Each model runs in its own isolated Docker container, launches its own IsaacSim instance, consumes standardized USD environments, and outputs trajectory data and metrics. No HAL integration is required. Extreme Parkour consumes depth observations from the collision mesh geometry. Holosoma runs proprioception-only (no vision) — it uses joint states and IMU as observations, providing a baseline that tests pure locomotion capability on the reconstructed terrain without relying on perception.
#### Acceptance Criteria
- Dockerfiles for Extreme Parkour and Holosoma build and run successfully; each container launches IsaacSim independently.
- Both models load the same set of USD environments (reconstructed) without path or asset errors.
- Extreme Parkour receives depth observations from the environment collision mesh; Holosoma runs proprioception-only. Both output trajectory and metric data in a consistent format.

### Task 4 - Hexapod Adaptation
#### Narrative
Convert both Extreme Parkour and Holosoma from quadruped to hexapod embodiment. Update action spaces, observation spaces, and URDF/embodiment configs for the higher DOF. Add reward shaping to encourage stable gaits: penalize all-legs-simultaneous motion and encourage alternating leg groups (tripod bias) to avoid unstable emergent gaits. Holosoma already has a clean extension pattern for adding new robots (see Milestone 5), so the hexapod adaptation should follow the same `holosoma_ext` registration flow.
#### Acceptance Criteria
- Action and observation spaces updated for hexapod DOF in both models; URDF/embodiment configs point to the Krabby hexapod asset.
- Reward shaping added penalizing simultaneous leg motion and encouraging tripod-style alternation; reward terms documented.
- Both models demonstrate stable hexapod locomotion in at least one environment without falls over a defined time window.

## Information
- The robot uses depth + PPO for locomotion. Visual fidelity (textures, lighting, Gaussian splatting) is not required. Collision mesh accuracy is the priority.
- Primary pipeline: COLMAP SfM → COLMAP MVS dense reconstruction → Poisson/Delaunay meshing → mesh conditioning → USD with physics.
- NVIDIA NuRec reference workflow (COLMAP + 3DGUT) is the starting point, but 3DGUT Gaussian training is optional since we don't need photorealistic RGB rendering.
- Scale calibration is critical: monocular reconstruction has inherent scale ambiguity. Include a known-size reference object in captures or measure post-reconstruction.
- Capture guidelines: ~60% photo overlap, multiple heights/angles, locked focus/exposure, avoid reflective surfaces / motion blur / low-texture regions. JPEG/PNG format (convert HEIC).
- Models in scope: Extreme Parkour (depth-based), Holosoma quadruped (proprioception-only, no vision). SoloParkour is deferred until its code repo is published (see Appendix C). LocoMamba is deferred.
- Non-goals: real-world deployment, HAL integration, robotic arm support, UI/dashboard, procedural environment generation (deferred), photorealistic rendering.
- Repository structure:
  ```
  krabby-research/
  ├── environments/
  │   └── reconstructed/
  ├── models/
  │   ├── extreme_parkour/
  │   └── holosoma/
  └── docker/
  ```

## FAQ
- Why COLMAP over SLAM3R for the primary pipeline?
  COLMAP is the industry-standard SfM/MVS pipeline with mature dense reconstruction and built-in meshing. It's the backbone of NVIDIA's NuRec workflow and GaussGym. SLAM3R is a newer feed-forward model that's faster but less proven for producing collision-quality meshes. See Appendix A for SLAM3R as an alternative.
- Why not use 3DGUT Gaussian splatting?
  The robot uses depth + PPO, not RGB. Gaussian splatting provides photorealistic rendering but no collision geometry. The collision mesh from COLMAP MVS + Poisson reconstruction is what the robot actually needs. 3DGUT can optionally be added later if RGB rendering becomes useful.
- Why is mesh conditioning mandatory?
  Raw reconstructions contain floaters, holes, bad topology, and non-walkable surfaces. Without conditioning, the robot's depth sensor sees garbage geometry and the physics simulation breaks.
- How do you handle scale ambiguity?
  Monocular reconstruction has no absolute scale. Either include a known-size reference object in the scene during capture, or measure a known real-world distance and apply a uniform scale correction post-reconstruction.
- What if the collision mesh is too noisy for locomotion?
  Retrain models on the reconstructed meshes to close the domain gap. Use simplified collision proxies (convex decomposition via V-HACD) for particularly problematic geometry. As a last resort, add a flat ground plane and use the mesh only for obstacle geometry.

## Appendix A — SLAM3R Alternative Pipeline (Reference)
If COLMAP proves too slow or the photo-based capture workflow is impractical, SLAM3R provides a video-first alternative. This was the original M11 approach before switching to the NuRec/COLMAP pipeline.

### Pipeline: SLAM3R → Open3D Poisson → Mesh Conditioning → USD
- Input: monocular video (phone capture), not photos
- SLAM3R (`github.com/PKU-VCL-3DV/SLAM3R`): two-hierarchy neural network (I2P for local reconstruction, L2W for global registration). Directly regresses dense 3D pointmaps from video at 20+ FPS on RTX 4090. Outputs globally aligned point cloud.
- MASt3R-SLAM (`github.com/edexheim/mast3r_slam`): fallback. Monocular dense SLAM using MASt3R 3D reconstruction priors, 15 FPS on RTX 4090.
- Spann3R: secondary fallback for feed-forward reconstruction.
- Meshing: export point cloud as PLY → Open3D `estimate_normals()` → `create_from_point_cloud_poisson(depth=9)` → density-based cropping.
- Conditioning + USD export: same as Task 2 (Open3D / PyMeshLab / trimesh → Isaac Lab `MeshConverter`).

### Trade-offs vs COLMAP
| | COLMAP | SLAM3R |
|---|---|---|
| Input | Photos (~60% overlap) | Video (continuous) |
| Speed | Slow (minutes–hours for MVS) | Fast (real-time feed-forward) |
| Maturity | Industry standard, battle-tested | Newer research, less proven |
| Dense reconstruction | Built-in MVS pipeline | Direct pointmap regression |
| Mesh quality | High (MVS + Poisson/Delaunay) | Depends on pointmap density |
| NVIDIA ecosystem | Native (NuRec workflow) | Requires manual integration |
| Scale | Ambiguous (monocular) | Ambiguous (monocular) |

### When to use SLAM3R instead
- Video capture is more natural than taking hundreds of photos (e.g., walking through a space)
- Faster iteration is needed (no MVS compute time)
- COLMAP fails on the scene (e.g., too few features, repetitive textures)

## Appendix B — 3DGUT / NuRec Visual Layer (Optional)
If depth-only perception proves insufficient and RGB observations become needed (e.g., for semantic navigation or visual PPO), the 3DGUT Gaussian splatting layer can be added on top of the collision mesh pipeline.

### Adding 3DGUT
- Requires: COLMAP sparse reconstruction (already produced in Task 0)
- Run: `python train.py --config-name apps/colmap_3dgut_mcmc.yaml path=/path/to/colmap/ export_usdz.enabled=true`
- Output: USDZ with Gaussian splatting data, loadable in Isaac Sim 5.0+ as a neural volume
- The Gaussian layer provides photorealistic RGB rendering; the collision mesh (from Task 1–2) provides physics
- GaussGym (`gauss-gym.com`) demonstrates this dual approach: Gaussians for rendering, separate collision mesh for physics, achieving 100K+ steps/sec in IsaacGym

### When to add 3DGUT
- Robot policy needs RGB observations (not just depth)
- Semantic navigation tasks (e.g., avoid puddles, follow paths) where depth alone is insufficient
- Demo/visualization purposes where photorealistic rendering is desired

## Appendix C — SoloParkour (Deferred)
SoloParkour is a strong candidate for future inclusion but its code repository is not yet published. Once available, it would replace or supplement Holosoma as the second evaluation model.

### Integration plan (when repo is available)
- Runs in its own Docker container like Extreme Parkour
- Consumes depth observations from the collision mesh
- Same hexapod adaptation process (action/obs space update, reward shaping)
- Same evaluation metrics for direct comparison

### Why it was deferred
- Code repo not published — cannot build, containerize, or adapt without source access
- Holosoma provides a viable proprioception-only baseline in the meantime, and its extension pattern is already proven from Milestone 5
