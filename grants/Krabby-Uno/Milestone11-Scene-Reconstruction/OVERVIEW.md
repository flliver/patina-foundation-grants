# Patina Foundation Grant - Krabby-Uno Milestone 11: Scene Reconstruction & Locomotion Benchmarking

## Grant Overview
Build a simulation pipeline that converts monocular video into IsaacSim environments and evaluates locomotion models on them. The pipeline uses SLAM3R to produce globally aligned point clouds from phone video, conditions and converts them to USD meshes, and imports into IsaacSim. Two locomotion models (Extreme Parkour and SoloParkour) are then evaluated on identical reconstructed environments using a hexapod embodiment, producing comparable metrics and demo videos.

## Why is this Important?
- Bridges the real-to-sim gap by turning ordinary phone video into walkable simulation environments, making environment creation accessible to anyone with a camera.
- Provides an apples-to-apples comparison framework for locomotion models on identical environments, enabling rigorous benchmarking.
- Extends state-of-the-art quadruped locomotion models to hexapod embodiment, broadening the research landscape for multi-legged robots.
- Explicitly addresses mesh conditioning (noise removal, hole filling, collision proxy generation), the step most reconstruction pipelines skip and the reason most sim transfers fail.

## Tasks
### Task 0 - SLAM3R Video Reconstruction (Video → Point Cloud)
#### Narrative
Capture monocular video of 2–3 indoor/outdoor spaces using a phone and run SLAM3R to produce globally aligned dense point clouds. SLAM3R is a two-hierarchy neural network (I2P for local reconstruction within sliding windows, L2W for global registration) that directly regresses 3D pointmaps from monocular RGB video at 20+ FPS on an RTX 4090. It is chosen over SAM3D because it reconstructs globally consistent environments rather than object-level Gaussian splats. MASt3R-SLAM is an acceptable alternative if SLAM3R proves difficult to set up. Export the resulting point clouds as PLY files for downstream meshing.
#### Tooling
- SLAM3R (`github.com/PKU-VCL-3DV/SLAM3R`): primary reconstruction model. Accepts video input, outputs dense pointmaps in global coordinates via sliding-window I2P + L2W alignment.
- MASt3R-SLAM (`github.com/edexheim/mast3r_slam`): fallback. Monocular dense SLAM leveraging MASt3R 3D reconstruction priors, 15 FPS on RTX 4090, globally consistent poses + dense geometry.
- Spann3R: secondary fallback for feed-forward reconstruction.
- Export: save point clouds as PLY using Open3D (`open3d.io.write_point_cloud`).
#### Acceptance Criteria
- Phone video captured following guidelines: slow smooth motion, full loop (start/end same position), avoiding reflective surfaces / motion blur / low-texture regions.
- SLAM3R (or fallback) produces a dense point cloud in a global coordinate frame from each video; PLY files exported and committed.
- Point clouds are visually inspectable in Open3D or MeshLab; coverage and alignment verified across 2–3 scenes.

### Task 1 - Point Cloud to Mesh Conversion (Point Cloud → OBJ)
#### Narrative
Convert the SLAM3R point clouds into triangle meshes suitable for simulation. Use Open3D's Poisson surface reconstruction (`create_from_point_cloud_poisson`) as the primary method — it produces watertight meshes from oriented point clouds and handles the density variations typical of SLAM output. If normals are missing, estimate them with `estimate_normals`. After Poisson reconstruction, crop low-density regions using the density filter to remove extrapolated geometry. Export as OBJ for maximum compatibility with downstream tools.
#### Tooling
- Open3D (`open3d`): primary library for the full pipeline.
  - `open3d.geometry.PointCloud.estimate_normals()`: compute normals if SLAM3R output lacks them.
  - `open3d.geometry.TriangleMesh.create_from_point_cloud_poisson(pcd, depth=9)`: Poisson surface reconstruction. `depth` controls resolution (8–10 typical for room-scale).
  - Density-based cropping: remove vertices below a density percentile threshold to trim Poisson extrapolation artifacts.
- PyMeshLab (`pymeshlab`): alternative if Poisson in Open3D underperforms. Offers screened Poisson, ball-pivoting, and marching cubes via MeshLab filters.
- Export: `open3d.io.write_triangle_mesh("scene.obj", mesh)` or `trimesh.exchange.export.export_mesh`.
#### Acceptance Criteria
- Normals estimated (if needed) and Poisson reconstruction produces a watertight triangle mesh from each point cloud.
- Low-density extrapolation artifacts cropped; mesh visually matches the original point cloud coverage.
- Meshes exported as OBJ files; loadable in MeshLab/Blender for visual inspection.

### Task 2 - Mesh Conditioning & USD Export (OBJ → USD)
#### Narrative
Raw reconstructed meshes are not walkable. Apply mandatory conditioning: statistical outlier removal, hole filling, Laplacian/Taubin smoothing, quadric-edge-collapse decimation, and collision proxy generation. Then convert to USD with corrected scale (monocular ambiguity), Z-up orientation, and origin alignment. Import into IsaacSim and verify the environment loads. Support two collision modes: visual fidelity (full mesh + flat ground plane or simplified collision mesh) and physical interaction (clean collision mesh with mesh collider).
#### Tooling
- Open3D:
  - `remove_degenerate_triangles()`, `remove_unreferenced_vertices()`: basic cleanup.
  - `filter_smooth_laplacian()` / `filter_smooth_taubin()`: surface smoothing (Taubin avoids shrinkage).
  - `simplify_quadric_decimation(target_number_of_triangles)`: mesh decimation.
- PyMeshLab (`pymeshlab`): heavier-duty conditioning.
  - `remove_isolated_pieces_wrt_diameter()`: floater/outlier removal.
  - `close_holes(maxholesize=N)`: hole filling.
  - `simplification_quadric_edge_collapse_decimation()`: decimation with quality preservation.
  - `generate_simplified_point_cloud()` → convex hull for collision proxy.
- trimesh (`trimesh`):
  - `trimesh.repair.fill_holes()`, `trimesh.repair.fix_winding()`, `trimesh.repair.fix_normals()`: quick repair pass.
  - `trimesh.convex.convex_hull(mesh)`: generate convex collision proxy.
  - `trimesh.decomposition.convex_decomposition(mesh)`: V-HACD approximate convex decomposition for non-convex collision shapes.
- USD conversion:
  - Isaac Lab `MeshConverter` (`isaaclab.sim.converters.mesh_converter`): converts OBJ/STL/FBX → USD with physics properties, collision meshes, and instanceable format. Preferred path for IsaacSim integration.
  - `omni.kit.asset_converter` (Omniverse Kit extension): batch conversion of OBJ/FBX/glTF → USD.
  - Manual: `pxr` (OpenUSD Python API) for fine-grained control over `UsdGeom`, `UsdPhysics`, and `PhysxSchema` if the converters don't handle collision modes correctly.
#### Acceptance Criteria
- Conditioning pipeline removes floaters/noise, fills holes, smooths surfaces, and decimates; before/after comparison documented.
- Collision mesh generated for each environment (visual fidelity or physical interaction mode selectable).
- USD export has correct scale, Z-up orientation, and origin; each environment loads in IsaacSim without errors.

### Task 3 - Locomotion Model Integration (Dockerized)
#### Narrative
Integrate Extreme Parkour and SoloParkour as the two evaluation models. Each model runs in its own isolated Docker container, launches its own IsaacSim instance, consumes standardized USD environments, and outputs trajectory data and metrics. No HAL integration is required.
#### Acceptance Criteria
- Dockerfiles for Extreme Parkour and SoloParkour build and run successfully; each container launches IsaacSim independently.
- Both models load the same set of USD environments (reconstructed) without path or asset errors.
- Each model outputs trajectory and metric data in a consistent format for downstream evaluation.

### Task 4 - Hexapod Adaptation
#### Narrative
Convert both Extreme Parkour and SoloParkour from quadruped to hexapod embodiment. Update action spaces, observation spaces, and URDF/embodiment configs for the higher DOF. Add reward shaping to encourage stable gaits: penalize all-legs-simultaneous motion and encourage alternating leg groups (tripod bias) to avoid unstable emergent gaits.
#### Acceptance Criteria
- Action and observation spaces updated for hexapod DOF in both models; URDF/embodiment configs point to the Krabby hexapod asset.
- Reward shaping added penalizing simultaneous leg motion and encouraging tripod-style alternation; reward terms documented.
- Both models demonstrate stable hexapod locomotion in at least one environment without falls over a defined time window.

## Information
- Primary reconstruction tool: SLAM3R (MASt3R-SLAM or Spann3R as fallbacks). SAM3D is not suitable as the primary pipeline due to lack of global alignment and simulation-ready geometry, but may optionally augment scenes with semantic objects.
- Mesh pipeline: Open3D (Poisson reconstruction, smoothing, decimation) + PyMeshLab (hole filling, floater removal) + trimesh (repair, convex decomposition for collision).
- USD conversion: Isaac Lab `MeshConverter` (preferred) or `omni.kit.asset_converter` for OBJ → USD with physics/collision properties.
- Capture guidelines: slow smooth camera motion, full loop (start/end same position), avoid reflective surfaces, motion blur, and low-texture regions.
- Known reconstruction issues (not bugs): scale ambiguity, floaters/artifacts, baked lighting, missing geometry — all inherent to monocular reconstruction.
- Models in scope: Extreme Parkour, SoloParkour only. Holosoma and LocoMamba are deferred (additional embodiment work and training pipeline alignment needed).
- Non-goals: real-world deployment, HAL integration, robotic arm support, UI/dashboard, procedural environment generation (deferred).
- Repository structure:
  ```
  krabby-research/
  ├── environments/
  │   └── reconstructed/
  ├── models/
  │   ├── extreme_parkour/
  │   └── solo_parkour/
  └── docker/
  ```

## FAQ
- Why SLAM3R over SAM3D?
  SAM3D excels at object-level reconstruction but outputs Gaussian splats / partial meshes without global scene alignment. SLAM3R reconstructs globally consistent environments directly from video sequences, producing dense pointmaps aligned in world space.
- Why is mesh conditioning mandatory?
  Raw reconstructions contain floaters, holes, bad topology, and non-walkable surfaces. Without conditioning, no locomotion model can traverse them.
- What if models fail on reconstructed environments?
  Retrain on reconstructed meshes or a mixed dataset. Support simplified collision proxies (convex decomposition via trimesh/V-HACD) if meshes remain non-walkable.
- What collision modes are supported?
  Visual fidelity mode (full mesh + flat ground plane or simplified collision mesh) and physical interaction mode (clean collision mesh with IsaacSim mesh collider).
- Why Open3D + PyMeshLab + trimesh instead of one library?
  Open3D is strongest at point cloud processing and Poisson reconstruction. PyMeshLab has the best hole-filling and floater removal filters. trimesh excels at repair and convex decomposition for collision proxies. The three complement each other well.
- How does USD conversion work?
  Isaac Lab's `MeshConverter` wraps `omni.kit.asset_converter` and outputs instanceable USD with physics properties and collision meshes — the format IsaacSim expects for learning environments.
