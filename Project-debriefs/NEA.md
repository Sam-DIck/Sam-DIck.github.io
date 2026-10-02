# A-Level Computer Science NEA Project
## Overview
As part of my A-Level Computer Science coursework, I created a basic game boat fishing game. The game allows players to control a boat, navigate a lake, and catch fish.

During this project, I focused on realistic physics simulation for the boat's buoyancy and the behavior of fish within the lake environment.

## Key Features
- Fish schooling behavior using GPU compute shaders
- Realistic boat buoyancy simulation
- Procedurally generated lake terrain
## Technologies
- Unity 3D
- Languages: C#, HLSL
- GPU Compute Shaders
- k-tree spatial partitioning


## 10,000 Fish in real-time

I developed a real-time fish schooling simulation based on an adapted version of Craig Reynolds' **Boids** algorithm.

The project focused primarily on scalability: starting with a straightforward CPU implementation, I progressively redesigned the system to support increasingly large schools by addressing the main computational bottlenecks.

The final GPU-based implementation was capable of simulating approximately **10,000–15,000 fish** in real time.

### The Initial Implementation

The first implementation ran entirely on the CPU.

Each fish evaluated its relationship with every other fish to determine the local forces required for schooling behaviour, such as:

* **Separation** — avoiding nearby fish
* **Alignment** — matching the velocity of neighbours
* **Cohesion** — moving towards the local group

This resulted in an \(O(n^2)\) algorithm: every fish potentially needed to inspect every other fish.

While straightforward to implement, the approach scaled poorly. In practice, I found that approximately **10–15 fish** could be simulated before the computational cost became noticeable.

This made the primary optimisation target clear: **neighbour searching**.

### Spatial Partitioning

The second implementation introduced a spatial partitioning structure based on a **k-d tree**.

Instead of every fish checking every other fish, the environment was divided spatially. A fish only needed to search its own region and relevant neighbouring regions when looking for nearby fish.

This significantly reduced the amount of unnecessary distance checking and allowed the simulation to scale to approximately **500 fish** while maintaining acceptable performance.

This iteration demonstrated an important principle of optimisation: rather than optimising the individual distance calculation, I changed the algorithm so that most distance calculations no longer needed to happen at all.

### Moving the Simulation to the GPU

The next bottleneck was the sheer number of independent calculations being performed for each fish.

Because the schooling calculations are highly parallel, I moved the simulation to the GPU using **compute shaders**.

I initially investigated combining the GPU implementation with the spatial partitioning approach, but implementing a suitable GPU-based k-d tree proved more complex than anticipated. Instead, I returned to the simpler neighbour-search algorithm and focused on exploiting the GPU's parallelism.

#### The final implementation used a **two-pass compute pipeline**.

#### Pass 1 — Calculate Acceleration

The first compute pass read each fish's position and velocity and calculated its resulting acceleration from the schooling rules.

The acceleration for each fish was written into a separate buffer.

Keeping this calculation separate from the integration step avoided race conditions caused by one fish reading data that another fish was simultaneously modifying.

#### Pass 2 — Integrate Motion

The second pass applied the calculated accelerations to update each fish's velocity and position.

Conceptually:

$$
a_i = f(\text{neighbours}_i)
$$

followed by:

$$
v_i \leftarrow v_i + a_i \Delta t
$$

$$
p_i \leftarrow p_i + v_i \Delta t
$$

Separating these operations into two passes ensured that every fish was working from a consistent snapshot of the previous simulation state.

### Results

The three implementations provided a useful progression in scalability:

| Implementation | Approach                                      |       Approx. Scale |
| -------------- | --------------------------------------------- | ------------------: |
| CPU            | Naive all-to-all search                       |          10–15 fish |
| CPU            | Spatial partitioning / k-d tree               |           ~500 fish |
| GPU            | Compute shader, parallelised neighbour search | ~10,000–15,000 fish |

The final system could therefore simulate **orders of magnitude more fish** than the original implementation.

The project also gave me practical experience with a common GPU-compute constraint: moving a calculation to the GPU is not automatically enough. Data movement between CPU and GPU can itself become the bottleneck.

### GPU ↔ CPU Data Transfer

One of the limitations of the final implementation was the boundary between the simulation and rendering systems.

Fish positions, velocities, and accelerations were copied out of GPU buffers and processed on the CPU to construct the **TRS matrices** required for indirect instanced rendering.

This introduced CPU/GPU synchronisation and data-transfer overhead.

At sufficiently large fish counts, the cost of transferring and preparing this data became a significant portion of the total workload.

A more advanced implementation would keep the entire pipeline GPU-resident:

$$
\text{Simulation}
\rightarrow
\text{Transforms}
\rightarrow
\text{Indirect Rendering}
$$

without requiring the CPU to retrieve the simulation state each frame.

At the time, I had not yet encountered this fully GPU-resident approach; discovering it later highlighted that the optimization problem extended beyond the compute shader itself and into the architecture of the rendering pipeline.

### Limitations

- **Neighbour Search**

    Although the GPU implementation parallelised the workload, each fish still performed an \(O(n)\) neighbour search.

    This meant that increasing the school size eventually caused the neighbour-search cost to dominate again.

    A GPU-friendly spatial acceleration structure, such as a uniform grid or GPU-built spatial hash, would allow the search to scale more effectively.
    
- **CPU/GPU Synchronisation**

    The simulation state was transferred back to the CPU to generate rendering transforms.

    This introduced unnecessary synchronisation and data movement. Keeping the simulation and rendering data entirely on the GPU would remove this bottleneck.

- **No Environmental Awareness**

    The fish only considered other fish when determining their movement.

    They had no awareness of:

    * Terrain
    * The lake bed
    * Obstacles
    * The water surface
    * Other environmental geometry

    As a result, fish could swim through objects, intersect the terrain, or leave the water entirely.

    A more complete implementation could incorporate environment-aware steering, collision avoidance, or a GPU-friendly representation of the surrounding volume.

### Outcome

The project evolved from a simple CPU implementation into a GPU-accelerated simulation capable of handling **thousands of independently simulated agents**.

The most valuable part of the project was the optimisation process itself: profiling the initial implementation, identifying neighbour searching as the dominant cost, experimenting with spatial partitioning, and eventually restructuring the simulation around massively parallel GPU computation.

It also exposed an important distinction between **GPU computation** and a genuinely **GPU-native pipeline**. Although the final simulation performed its calculations on the GPU, CPU/GPU data transfers remained a significant bottleneck, providing a clear direction for further optimisation.

## Making Things Float

I developed a mesh-based buoyancy system capable of applying buoyant forces to arbitrary 3D objects, including concave meshes and objects that are only partially submerged.

The goal was not to produce a physically perfect fluid simulation, but a stable and convincing approximation that could respond naturally to changing conditions. The resulting system allowed objects to float, tilt under changing loads, and respond to external forces such as wind, motors, and player movement.

### From Arbitrary Meshes to Volume

The first problem was calculating the volume of an arbitrary mesh.

Since a mesh is composed of triangles, I decomposed each triangle into a tetrahedron using a common reference point. I experimented with several choices for this point, including the average of all vertices and the centre of the bounding box. The resulting volume differences were relatively small for the meshes I was targeting, so I chose the bounding-box centre for its simplicity and low computational cost.

For a tetrahedron defined by the reference point \(M\) and triangle vertices \(v_1, v_2, v_3\), its signed volume can be calculated using the scalar triple product:

$$
V = \frac{1}{6}
\vec{v_1M}
\cdot
\left(
\vec{v_1v_2}
\times
\vec{v_1v_3}
\right)
$$

The important detail here is that the volume is **signed** rather than taking its absolute value.

This turned out to be useful rather than problematic. Correctly wound triangles contribute positive volume while oppositely oriented geometry contributes negative volume. As a result, overlapping or intentionally reversed regions of a mesh could effectively cancel out, allowing the same approach to handle more complex and concave geometry without requiring the mesh to be explicitly converted into a collection of convex pieces.

### From Volume to Buoyancy

Once the volume of each tetrahedron could be determined, Archimedes' principle provided the basis for calculating buoyancy:

$$
F_b = \rho V g
$$

where:

* \(F_b\) is the buoyant force
* \(\rho\) is the density of the fluid
* \(V\) is the submerged volume
* \(g\) is gravitational acceleration

Rather than applying a single force to the object's centre of mass, I calculated the contribution of each tetrahedron independently and applied the resulting force at that tetrahedron's centre.

This distributed approach was important. It meant that buoyancy naturally generated **torque** as the object rotated or as its mass distribution changed, rather than simply pushing the object upwards.

### Handling Partial Submersion

The next problem was that the full tetrahedron volume should only contribute while submerged.

For each tetrahedron, I first checked whether it was completely above the water surface. If so, it could be discarded immediately.

For tetrahedra intersecting the water surface, I projected the relevant vertices onto the water surface and used the resulting geometry to approximate the submerged volume.

This was deliberately an approximation rather than a true geometric clipping operation. At the time, accurately clipping every tetrahedron against a dynamically displaced wave surface was outside the scope of the project.

The approximation was generally convincing, but became less reliable with particularly unusual geometry or aggressive wave shapes.

### Applying Forces

The calculated buoyant force was applied through Unity's physics system at the centre of each tetrahedron.

This produced an emergent behaviour that was more interesting than simply making an object float:

* Objects naturally tilted as their submerged volume changed.
* Moving mass affected the object's orientation.
* External forces could interact with buoyancy rather than being overridden by it.
* Concave or negatively oriented regions could contribute opposing forces.
* Multiple points of force application produced realistic-looking rotational behaviour.

This made it possible for a player character to walk around on a floating object and visibly affect its attitude as their weight moved around the surface.

The same system could also respond to forces such as wind or propulsion from a motor.

### Limitations

The system deliberately traded physical accuracy for computational simplicity.

- **Approximate water clipping**

    The submerged geometry was approximated by projecting vertices rather than explicitly clipping each tetrahedron against the water surface.
    This worked well for typical geometry, but could produce inaccurate volumes when a tetrahedron intersected a strongly curved or rapidly changing wave.

- **Mesh resolution**

    Because the calculation was based on the mesh's existing triangles, low-poly geometry could produce poor approximations of the actual submerged volume.
    For example, a tetrahedron could span a wave crest and trough while having all of its vertices on opposite sides of the surface. The system therefore inherited some of the limitations of the input mesh.

- **Vertex winding**

    The signed-volume calculation assumes consistent triangle winding. Incorrectly wound geometry can therefore produce incorrect or opposing volume contributions.

- **Physical approximation**

    The system was designed to produce stable, believable behaviour rather than solve the full fluid-dynamics problem. Effects such as water displacement, drag, turbulence, and wave interaction were simplified or omitted.

### Outcome

Despite these limitations, the system produced stable and convincing buoyancy across arbitrary mesh geometry.

More importantly, the distributed-force approach made the simulation responsive to the rest of the physics system. Objects did not simply move vertically in response to buoyancy; they reacted to where forces were being applied and how their mass was distributed.
