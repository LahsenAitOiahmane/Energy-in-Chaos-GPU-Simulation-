# Energy in Chaos: GPU-Accelerated Lagrangian Particle Advection

**Author:**  
[Your Name / Entity]

**Tech Stack:**  
WebGL2 • Three.js • GLSL • GPGPU

**License:**  
MIT

---

## 1. Abstract

**Energy in Chaos** is a real-time, interactive visualization of a turbulent flow-field simulation running entirely on the GPU.  
By using **GPGPU (General-Purpose computing on Graphics Processing Units)** techniques, the system can animate up to **10⁶ particles at 60 FPS in a web browser**.

Instead of solving the full Navier–Stokes equations, the simulation employs **Curl Noise potentials** to generate **divergence-free velocity fields**, producing fluid-like motion with drastically lower computational cost.

---

## 2. Theoretical Framework

### 2.1 Divergence-Free Velocity Fields

To mimic an incompressible fluid, the simulation must obey the constraint:

\[
\nabla \cdot \vec{v} = 0
\]

Standard Perlin/Simplex noise does not satisfy this and produces sinks/sources.  
To fix this, a **Curl Noise** field is constructed using a vector potential:

\[
\vec{v} = \nabla \times \vec{\psi}
\]

By vector calculus identity:

\[
\nabla \cdot (\nabla \times \vec{\psi}) = 0
\]

This guarantees **zero divergence**, producing natural swirling vortices and circulation patterns characteristic of real fluids.

The idea originates from the **Helmholtz–Hodge decomposition**, ensuring the velocity field is purely solenoidal.

---

### 2.2 Numerical Integration

The simulation follows a **Lagrangian** (particle-based) model.  
Particle positions are updated using **semi-implicit Euler integration**:

\[
P_{t+\Delta t} = P_t + \vec{v}(P_t,t)\Delta t
\]

Particles sample the curl noise velocity field at their current location.

---

## 3. Implementation Details

### 3.1 GPGPU Architecture

WebGL cannot read and write to the same texture in a single pass, so the system uses **Ping-Pong Buffers**.

All particle state is stored inside **floating-point textures (RGBA32F)**:

- **Position Texture**
  - R = x  
  - G = y  
  - B = z  
  - A = life  

- **Velocity Texture**
  - R = vx  
  - G = vy  
  - B = vz  
  - A = decay_rate  

With texture size \( N \times N \), particle count becomes:

\[
N^2
\]

Example:  
- \(1024^2 = 1,048,576\) particles  
- \(700^2 \approx 490,000\) particles  

Each frame:
1. Read old state from buffer A  
2. Compute next state in a fragment shader  
3. Write to buffer B  
4. Swap A and B  

---

### 3.2 Shader Kernels

#### **fragmentShaderVelocity**
- Samples 3D simplex noise  
- Computes numerical partial derivatives to approximate the curl  
- Generates divergence-free velocity  
- Applies:
  - Audio reactivity  
  - Mouse interaction forces  
  - Velocity blending (drag/inertia)

#### **fragmentShaderPosition**
- Integrates position: `pos += vel * delta`  
- Updates life/decay  
- Respawns particles inside a sphere when `life < 0`  

The entire physics system runs in the GPU’s fragment pipeline.

---

### 3.3 Instanced Rendering Pipeline

Rendering up to 1 million particles requires **Instanced Rendering** via `THREE.InstancedBufferGeometry`.

**Geometry:** one quad (two triangles).  
**Instancing:** GPU draws it \(10^6\) times in a single draw call.

Each instance contains a **reference UV** into the position/velocity textures:


The **vertex shader**:
- Samples particle position & velocity  
- Builds a **camera-facing billboard**  
- Applies **velocity-based stretching** for motion-blur appearance  

---

### 3.4 Post-Processing (HDR + Bloom)

Particles are rendered with **HDR brightness** (>1.0).  
A multi-pass **UnrealBloomPass** extracts bright regions and spreads them across the frame, creating:
- Neon-like glow  
- Plasma effects  
- Intense energetic highlights  

---

## 4. Performance Scaling

The simulation supports dynamic presets:

| Preset | Texture Size | Particle Count | Target Hardware |
|--------|--------------|----------------|-----------------|
| **Low** | 256×256 | 65,536 | Mobile / iGPU |
| **Medium** | 512×512 | 262,144 | Entry-level dGPU |
| **High** | 700×700 | ~490,000 | Mid-range PC |
| **Ultra** | 1024×1024 | 1,048,576 | High-end GPUs (RTX 3060+) |

Changing the preset reallocates GPU buffers in real time.

---

## 5. Dependencies

- **Three.js (r160)** — Rendering engine  
- **GPUComputationRenderer** — GPGPU simulation helper  
- **UnrealBloomPass** — Post-processing bloom  
- **lil-gui** — Live parameter controls  

---

