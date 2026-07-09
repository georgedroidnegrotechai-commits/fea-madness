# fea-madness

**Experimental Finite Element Analysis "Madness" – Alternative Meshes, Cutting-Edge Solvers & Macroscopic Molecule Vibes**

A BSD-3-Clause licensed C++ playground for pushing your in-house tet4/tet10 FEA (static stress, modal, thermal) beyond the ordinary. Explore hex-dominant, polyhedral/Voronoi, CutFEM, better solvers (iterative + preconditioners), automatic feature-aware meshing with smart transitions, and experimental discretizations that feel more like "macroscopic molecules" (isotropic, natural cell shapes) instead of rigid tets.

Built with Eigen::SparseLU as starting point, designed to be modular so you can experiment without fear. Goal: improve accuracy & performance of *your* simulations while having fun with research-grade ideas that have been done in academia but rarely in simple in-house codes.

Repo created live during our discussion on 2026-07-09. This README captures the full conversation summary, options analysis, proposed project spec, and a ready-to-use prompt you can feed to another Grok (or continue here) to start implementing the core.

## Table of Contents
- [FEA Meshes 101: What’s Possible & Is Your Current Already “Solid 3D”?](#fea-meshes-101-whats-possible--is-your-current-already-solid-3d)
- [Tet vs Everything Else: Pros, Cons & When to Switch](#tet-vs-everything-else-pros-cons--when-to-switch)
- [Better Meshers & Solvers for Your In-House Code](#better-meshers--solvers-for-your-in-house-code)
- [“Macroscopic Molecule” Shaped Elements – Options & Reality Check](#macroscopic-molecule-shaped-elements--options--reality-check)
- [Automatic Surface/Feature Transitions – What to Implement & Where](#automatic-surfacefeature-transitions--what-to-implement--where)
- [Project Vision & Proposed Architecture](#project-vision--proposed-architecture)
- [Initial Project Skeleton (Already in Repo)](#initial-project-skeleton-already-in-repo)
- [The Build Prompt – Copy-Paste to Start Implementation](#the-build-prompt--copy-paste-to-start-implementation)
- [Next Steps & How to Contribute/Experiment](#next-steps--how-to-contributeexperiment)
- [License](#license)

## FEA Meshes 101: What’s Possible & Is Your Current Already “Solid 3D”?

FEA supports a hierarchy of meshes:

- **1D**: Truss, beam, frame elements (lines with axial/bending DOFs).
- **2D**: Plane stress/strain, plate, shell – triangles (tri3/tri6) or quads (quad4/quad8/quad9).
- **3D Solid**: The ones you care about for “solid” parts.
  - **Tetrahedral**: tet4 (linear), tet10 (quadratic) – your current. Most flexible for automatic meshing of complex/watertight geometry from CAD/STL.
  - **Hexahedral**: hex8 (linear), hex20/serendipity (quadratic) – preferred for accuracy, convergence rate, bending, incompressibility (less volumetric locking). Structured hex easy on simple geo; unstructured/hex-dominant much harder.
  - **Wedges/Pyramids**: transition elements between tet and hex regions.
  - **Polyhedral**: general n-faced cells. Powerful with special formulations (see VEM below).
  - **Voxel / Cartesian background + cuts**: immersed or embedded methods.

**Important clarification on your question**: Yes, proper solid 3D FEA models use *fully volumetric meshes* with individual 3D elements that have volume and fill the interior of the part. The “edge/boundary representation” (B-rep from CAD, or surface triangles) is only the *input geometry* or the *surface mesh*. The mesher then creates nodes and connectivity for 3D elements *inside* the volume too. Your tet4/tet10 implementation already does this – it’s not surface-only. If it were only boundary elements it would be a shell/surface model, not solid. So you’re already working with true 3D solid elements.

The difference people mean by “just boundary” is when using shell elements on thin structures or when the volume mesh is poor quality ( sliver tets, bad aspect ratios). Good tet meshing still gives proper interior elements.

## Tet vs Everything Else: Pros, Cons & When to Switch

**Your tet4/tet10 + Eigen SparseLU is already a solid foundation.** Quadratic tet10 is *much* better than linear tet4 (better bending, less locking, faster convergence). Many production codes (Abaqus, Ansys, etc.) default to tets for complex parts precisely because robust automatic tet meshing exists (TetGen, Netgen, Gmsh, internal algos).

**Why hex is often “better”**:
- Higher accuracy per degree-of-freedom.
- Better convergence in bending and nearly incompressible materials.
- Less sensitive to element distortion.
- Structured hex can be very efficient.

**Downsides of hex**:
- Automatic generation for arbitrary geometry is still a research-grade problem (plastering, whisker-weaving, octree + pillowing, etc.). Most “auto hex” in commercial tools are hex-dominant (mostly hex + some tet/pyramid/wedge transitions) or require user guidance/swept regions.

**Hybrid is the practical sweet spot** for many: tet in complex regions, hex in mappable regions, pyramids/wedges for clean transitions.

**Polyhedral & advanced** open new doors (see below).

**Recommendation for you**: Keep tets as robust backbone. Add quality improvement tools first (smoothing, metrics, curvature-based sizing). Then experiment with hybrid or alternative discretizations in a sandbox module. Don’t throw away tet4/tet10 – enhance it and compare.

## Better Meshers & Solvers for Your In-House Code

### Mesh-side improvements (directly impact accuracy)
1. **Mesh quality tools** (critical & relatively easy win):
   - Jacobian checks, aspect ratio, dihedral angles, condition number.
   - Laplacian / optimization-based smoothing (node relocation).
   - Edge/face swapping on surface mesh.
   - Metrics + visualization (export bad elements to VTK colored by quality).
2. **Automatic sizing & feature detection** (see transitions section below).
3. **Importers**: Gmsh .msh (supports tet/hex/hybrid), perhaps STL surface + TetGen volume (if you add dep or wrapper).
4. **Experimental meshers**:
   - Simple octree/quadtree tet mesher (background grid refined near features).
   - Voronoi/Laguerre tessellation from point cloud or seed points inside volume → polyhedral cells that look more “natural/molecule-like”.

### Solver-side improvements (performance & scale)
Your current direct SparseLU (Eigen) is great for small-medium models (hundreds of thousands DOF). For bigger or many load cases:
- **Iterative solvers**: Eigen::ConjugateGradient (for SPD static stiffness) or BiCGSTAB/GMRES. Much lower memory, faster for very large sparse.
- **Preconditioners**: Diagonal/Jacobi (easy), incomplete Cholesky/LU (Eigen has some), or Algebraic Multigrid (AMG) – big win for scalability. Eigen doesn’t ship production AMG; options: implement simple smoothed aggregation, or later link external (Hypre via PETSc or custom, but adds deps).
- **Eigenvalue solvers for modal**: Eigen has limited; better wrap Spectra (header-only, ARPACK-like) or use shift-invert with iterative. For thermal eigenvalue (if needed) same.
- **Advanced**:
  - Matrix-free / on-the-fly assembly (no full K stored) for huge models.
  - Domain decomposition / FETI / BDDC for parallel.
  - GPU: cuSPARSE / cuSolver or Eigen CUDA backend (limited).

**Performance tip**: Profile assembly vs solve. Often assembly (especially numerical integration for tet10) can be optimized with better quadrature or vectorization.

### Cutting-edge / experimental to implement & benchmark
- **CutFEM / Immersed / Finite Cell Method**: Embed geometry in a simple background tet or hex grid (structured, easy to generate). “Cut” elements with the real boundary (level-set or ray casting on STL). Special quadrature rules + stabilization (ghost penalty or Nitsche) for accuracy. Huge advantage: no need to conform mesh to every design iteration or complex feature. Research very active, some production use in topology optimization.
- **Virtual Element Method (VEM)**: Allows *arbitrary polyhedral* elements (including Voronoi). Formulation looks like FEM but uses projection + stabilization. Excellent for complex meshes from Voronoi, image-based, or fractured domains. Accuracy comparable or better than tets in some tests. Papers + some open codes exist (e.g. in deal.II or research repos).
- **Isogeometric Analysis (IGA)**: Use NURBS/T-splines directly from CAD for exact geometry + higher continuity. Great accuracy but meshing = knot insertion, not traditional elements. Steeper learning curve.
- **Peridynamics or bond-based nonlocal models**: Discretization often looks like “molecules” with horizons/bonds between points. Excellent for fracture, no remeshing needed for cracks. Can prototype a simple bond-based solver and compare displacement/energy to classical FEA on same geometry.
- **Smoothed Finite Element / Edge-based / Face-based smoothed methods**: Improve accuracy of linear elements without going quadratic.

These have “been done” extensively in research (hundreds of papers). Some commercial codes have limited versions (e.g. LS-DYNA has some meshfree options, Abaqus has XFEM). Your in-house is perfect vehicle to experiment cheaply and see what actually helps *your* accuracy vs runtime on real parts.

## “Macroscopic Molecule” Shaped Elements – Options & Reality Check

You want elements that feel less like artificial sharp tets and more like rounded, natural, isotropic “macro molecules” or bubbles. This points toward:

- **Voronoi / Laguerre cells** (from random or optimized seed points inside volume): cells are convex polyhedra, often more equilateral/rounded than tets. When used with standard polyhedral FEM or (better) VEM, you get a mesh that visually and mechanically feels more “organic”. Several papers on Voronoi-based discretization for elasticity show good results, sometimes superior isotropy.
- **Laguerre Voronoi (power diagrams)** with weights can produce even more spherical-ish cells.
- **Peridynamics point clouds** with horizon: no traditional elements, just interacting “particles” with bonds – very molecule-like. Can be unstructured or lattice.
- **Meshfree methods** (Element-Free Galerkin, RKPM, SPH solids): nodes/particles with support domains, no connectivity. Can look spherical if radial basis or spherical supports used.
- **Radial basis or spherical harmonic enriched elements**: more exotic, possible but heavy.

**Feasibility**: Yes, prototypeable. Start with a simple point cloud inside your geometry → compute Voronoi (CGAL has it, or simple libraries, or even SciPy then import). Then either:
1. Use polyhedral assembly if you extend your element code (harder, need general poly quadrature or stabilization).
2. Or implement basic VEM (more elegant, literature has clear formulations for linear elasticity).

Many researchers have done exactly this for “natural” discretizations. It won’t replace tets overnight for robustness, but for specific parts or as comparison tool it’s gold for understanding.

## Automatic Surface/Feature Transitions – What to Implement & Where

Smart automatic handling of geometry features dramatically improves mesh quality and thus solution accuracy without user intervention. This is standard in good commercial meshers.

**What surfaces / features get what treatment**:

- **Circular / cylindrical / curved profiles (fillets, holes, pipes, spheres)**:
  - Curvature-based local refinement (smaller elements where radius small or curvature high).
  - Optional boundary-layer style inflation (prism or hex layers near surface for better gradient capture, then transition to tet inside). Useful even in pure structural for stress concentrations, contact, thermal gradients.
  - Circumferential node distribution control (avoid skinny elements around curves).
  - For perfect cylinders: swept or O-grid like structured patches if topology allows.

- **Straight edges, planar/rectangular faces, prismatic regions**:
  - Mapped or structured meshing where possible (great hex opportunity).
  - Uniform or slowly varying sizing.
  - Detect for possible multi-block decomposition or swept meshing.

- **Sharp corners, edges with high dihedral angle, vertices**:
  - Automatic feature preservation (don’t smooth away).
  - Local refinement (smaller h near singularity or stress riser).
  - Optional enrichment (singular elements or XFEM-like, advanced).
  - Quality checks to avoid degenerate tets at corners.

- **General transitions & size gradation**:
  - Smooth element size field (max growth rate ~1.5–2.0 between adjacent elements – prevents poor quality).
  - Pyramid or prism transition elements when mixing tet + hex regions.
  - Proximity-based refinement (thin walls, close features get finer mesh).
  - User seeds or curvature + feature size estimation from STL/CAD.

**Implementation path in fea-madness**:
- Geometry analyzer (STL or simple B-rep) that computes curvature (via normals or fitting), detects sharp features (dihedral angle threshold, e.g. > 30–45°), classifies faces (planar vs curved via normal variance or labels if CAD available).
- Sizing field generator (scalar or anisotropic) passed to mesher.
- Post-mesh quality improver + transition smoother.
- For hybrid: simple region detection or user-flagged “try hex here” zones.

Start simple: curvature + feature edge detection on surface mesh, then graded tet sizing. Later add boundary layer option and hybrid logic.

## Project Vision & Proposed Architecture

**Name**: fea-madness (fun, memorable, signals experimental nature).

**License**: BSD-3-Clause (chosen for maximum freedom to experiment and share).

**Core philosophy**: Your existing tet4/tet10 + direct solver as trusted core. Everything else is additive “mad science” modules you can toggle/compare. Validate everything against analytical solutions or known benchmarks first.

**Proposed high-level structure** (evolve as we build):

```
src/
  core/
    Mesh.hpp          # Nodes + connectivity, extensible element types
    DOFManager.hpp
    QualityMetrics.hpp  # Jacobian, aspect, etc. + bad element reporter
    GeometryUtils.hpp   # Curvature est, feature detection, sizing field
  elements/
    Tet4.hpp
    Tet10.hpp
    # later: Hex8, Hex20, Pyramid, Wedge, PolyVEM base
    ElementBase.hpp
  meshers/
    TetMesherWrapper.hpp   # your current or TetGen/Gmsh
    QualityImprover.hpp    # smoothing, swapping
    Experimental/
      OctreeMesher.hpp
      VoronoiMesher.hpp
      CutBackgroundGrid.hpp
  solvers/
    LinearSolver.hpp     # interface + SparseLU impl + CG + future AMG stub
    EigenvalueSolver.hpp # for modal (Spectra or Eigen based)
    # Thermal steady-state reuses linear solver
  physics/
    StaticStress.hpp
    ModalAnalysis.hpp
    Thermal.hpp
  utils/
    VTKExporter.hpp
    # logging, timing
experiments/ or examples/experiments/
  cutfem_prototype/
  voronoi_vem_comparison/
  peridynamics_toy/
```

**Tech stack**: Modern C++17/20, Eigen (core), CMake. Optional later: Spectra (header-only for eigensolve), CGAL (for Voronoi/geometry, heavy but powerful), VTK for viz (or simple export). Keep deps minimal at start so it’s easy to build on your machine.

**Validation first**: Every new feature must have a simple example (cantilever beam, pressurized cylinder, modal plate, steady thermal) with comparison to analytical or your old code.

## Initial Project Skeleton (Already in Repo)

We already created on your GitHub:
- LICENSE (BSD-3)
- README.md (this file)
- .gitignore (C++ CMake)
- CMakeLists.txt (finds Eigen, interface lib placeholder, examples/)
- examples/simple_demo.cpp + its CMake (builds a hello-world placeholder)

Clone it:
```bash
git clone https://github.com/georgedroidnegrotechai-commits/fea-madness.git
cd fea-madness
# Install Eigen if not present (apt, brew, or vcpkg, or FetchContent later)
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --target simple_demo_placeholder
./build/examples/simple_demo_placeholder
```

It will just print the placeholder message for now. Perfect starting point.

## The Build Prompt – Copy-Paste to Start Implementation

Here is a self-contained, detailed prompt you can give to another instance of Grok (or paste back here) to begin serious coding. It assumes the repo exists and the skeleton is there. It tells the AI exactly what to implement first, how to structure, what files to create/edit using tools if needed, and keeps scope reasonable for first iteration (make it compile + run a real simple tet4/tet10 validation case ported from your current code).

--- BEGIN BUILD PROMPT ---

You are an expert C++ FEA developer helping build "fea-madness", an experimental modular FEA framework. The GitHub repo https://github.com/georgedroidnegrotechai-commits/fea-madness already has initial skeleton (CMake, LICENSE, README, placeholder example).

**Task for this session**: Create a minimal but functional first implementation that ports the core of the user's existing tet4/tet10 static stress FEA (stiffness assembly + Eigen SparseLU solve) into the new structure. Make a working "hello world" validation: a simple 3D cantilever beam or unit cube under tension/shear/bending, using tet4 or tet10 elements, with basic BCs, solve for displacements, compute stress at a point or reaction forces, and compare qualitatively or to analytical where possible. Also add basic mesh data structures and quality stub.

**Strict requirements**:
- Use only Eigen as external dep (already in CMake).
- Modern C++17, clean headers, no raw new/delete where possible (smart pointers or containers).
- Keep it simple and extensible – no over-engineering.
- Everything must compile and run with the existing CMake after your changes.
- Add comments explaining where future extensions go (iterative solver, hex elements, CutFEM, VEM, Voronoi, feature detection).
- Export results to simple VTK ASCII or CSV for visualization (or note ParaView).
- At end, update the simple_demo.cpp or create a proper example that demonstrates the new functionality.
- Update README.md with a short "First Implementation" section describing what was done and how to run the new demo.
- Do **not** add heavy new dependencies.

**Step-by-step plan you should follow**:
1. Explore current repo state using tools if needed (get tree, read files).
2. Design minimal core classes:
   - `src/core/Mesh.hpp`: struct or class holding nodes (Eigen::Vector3d or Matrix), element connectivity (std::vector<std::vector<int>> or better indexed), element type tag or pointer to Element. Basic methods: numNodes(), numElements(), addNode, addElement, boundingBox, etc.
   - `src/core/DOFManager.hpp`: map nodes to DOFs (3 per node for solid), apply BCs (fixed DOFs), reduced system handling.
   - `src/elements/ElementBase.hpp` and `Tet4.hpp` / `Tet10.hpp`: virtual or CRTP for stiffness matrix computation (your existing integration code, or simple 1-point / 4-point quadrature for tet4, 4 or 5 point for tet10). Material (E, nu) or lame params. For tet10 also shape function derivatives.
   - `src/solvers/LinearSolver.hpp`: interface with solve(K, F) -> U. Concrete SparseLUSolver using Eigen::SparseLU. Later CG etc.
   - `src/physics/StaticStress.hpp`: class that takes Mesh + BCs + loads, assembles global K and F (loop over elements, call element->computeStiffness(), scatter), applies BCs via DOFManager, solves, recovers displacements/stresses.
3. Implement the above with placeholders or real simple code. For quadrature and shape functions you can hardcode standard tet formulas or port from user's knowledge.
4. Create a real example in examples/ : e.g. `cantilever_beam.cpp` that hardcodes a simple tet mesh (few elements or read simple .node/.ele files, or procedural cube mesh with tet subdivision), applies fixed end + tip load, runs StaticStress, prints tip displacement (compare to beam theory approx), exports VTK.
5. Make sure CMake picks up new src/ files (update CMakeLists.txt if needed to add sources or use target_sources, or make fea_madness_core a proper lib with add_library and glob or explicit sources).
6. Add a basic QualityMetrics stub that can compute min Jacobian or aspect for the mesh (even if dummy values first).
7. Test build + run. If issues, fix.
8. Finally, commit/push the changes via tools? (or describe the diffs and tell user the files changed).

**Output expectations**:
- All new .hpp/.cpp files created in proper dirs.
- Updated CMakeLists.txt if necessary.
- Working executable that solves a real small FEA problem.
- Updated README with usage and results example.
- Short discussion in response: what was easy/hard, ideas for next (iterative solver, quality tools, Voronoi experiment stub).

This is the foundation. Once done, the user can feed follow-up prompts for “add iterative CG solver option”, “implement basic feature detection + curvature sizing”, “add hex8 element stub”, “prototype simple CutFEM on background grid”, etc.

--- END BUILD PROMPT ---

Copy the entire block above (between the --- lines) and give it to a fresh Grok (or me again) together with any details about your current tet4/tet10 implementation (shape function code, assembly snippet, how you generate the tet mesh today, example geometry you validate on). The more specifics you provide about your existing code, the better the port will be.

## Next Steps & How to Contribute/Experiment

1. **Immediate**: Use the build prompt above to get a working port of your core solver into the new modular structure. This gives you a clean base to experiment from.
2. **Then iterate**:
   - Add your mesh quality metrics + simple smoother.
   - Implement curvature/feature detector + automatic sizing field for tet mesher.
   - Add ConjugateGradient solver option and benchmark vs SparseLU on larger meshes.
   - Prototype Voronoi point seeding + simple polyhedral or VEM assembly on a cube (compare isotropy/stress to tet mesh).
   - Try a toy CutFEM on a background tet grid with a spherical or complex inclusion.
3. **Longer term mad ideas**: Peridynamics comparison module, IGA toy, GPU solver experiments, adaptive h-refinement driven by error estimator, hybrid tet-hex with pyramid transitions, etc.
4. **Validation culture**: Every new discretization or solver must beat or match the old tet4/tet10 on standard benchmarks before claiming victory.
5. **Community**: Since BSD, feel free to share interesting experiment results, meshes, or papers that inspired a module. Open issues for ideas.

This project exists because you wanted to experiment seriously with alternatives while keeping control of your in-house code. Now you have a dedicated home for the madness.

Let’s make some beautiful (or interestingly ugly) meshes and see what they can do.

## License

BSD 3-Clause License – see LICENSE file. Free to use, modify, share, even commercially, with the attribution conditions.

---

*Built live with Grok on 2026-07-09. Questions, ideas, or ready to run the build prompt? Just say the word.*
