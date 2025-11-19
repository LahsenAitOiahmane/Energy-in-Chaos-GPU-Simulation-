# Energy in Chaos — GPU Particle Simulation

A real-time physically inspired particle simulation built with **Three.js**, **GPUComputationRenderer**, and custom **GLSL shaders**.  
The system simulates turbulent “energy fields” using **curl noise**, **simplex noise**, **GPGPU advection**, and **instanced rendering** to support hundreds of thousands of particles at high performance.

---

##  Overview

This visualization represents chaotic energy behaving like a living, fluid system.  
It uses:

- Divergence-free **curl noise** (turbulence)
- **3D simplex noise** (coherent noise field)
- **Particle advection** (velocity-driven motion)
- **Life/decay cycles**
- **GPGPU physics** (positions and velocities in floating-point textures)
- **Instanced billboards** for rendering
- **Bloom/post-processing**
- Optional **audio reactivity**  

Every particle is computed on the GPU each frame, enabling up to **1,000,000 particles**.

---

#  Physics Theory

## 1. Particle State

Each particle has:
- Position: `p = (x, y, z)`
- Velocity: `v = (vx, vy, vz)`
- Life scalar: `life ∈ [0,1]`

Motion uses:

\[
\frac{d\vec{p}}{dt} = \vec{v}
\]

\[
\vec{p}_{new} = \vec{p} + \vec{v}\Delta t
\]

---

## 2. Simplex Noise

The simulation uses a 3D **simplex noise** field:

\[
n = \text{snoise}(x,y,z)
\]

Simplex noise is smooth, continuous, and ideal for procedural turbulence.

---

## 3. Curl Noise (Divergence-Free Flow)

Curl noise is used to create **fluid-like swirling motion**.

Definition:

\[
\text{curl}(F) =
\begin{bmatrix}
\frac{\partial F_z}{\partial y} - \frac{\partial F_y}{\partial z} \\
\frac{\partial F_x}{\partial z} - \frac{\partial F_z}{\partial x} \\
\frac{\partial F_y}{\partial x} - \frac{\partial F_x}{\partial y}
\end{bmatrix}
\]

Properties:
- **Divergence free** → fluid-like  
- Produces natural vortices  
- Ideal for smoke, fire, magical plasma, energy FX  

The shader approximates these derivatives with finite differences.

---

## 4. Velocity Update (Advection)

Velocity is blended toward the curl noise field:

\[
\vec{v}_{target} = \text{curl}\left(\text{snoise}(p)\right)
\]

\[
\vec{v}_{new} = (1 - k)\vec{v} + k\vec{v}_{target}
\]

This produces **stable, swirling turbulence**.

---

## 5. Life & Decay

Each particle loses life:

\[
life = life - dieSpeed
\]

When life reaches zero, the particle respawns within a random sphere:

\[
\begin{aligned}
x &= r \sin(\phi)\cos(\theta) \\
y &= r \sin(\phi)\sin(\theta) \\
z &= r \cos(\phi)
\end{aligned}
\]

This generates a continuous system.

---

## 6. Mouse Interaction

Particles are repelled when near the cursor:

\[
\vec{F}_{mouse} =
\frac{\vec{p}-\vec{m}}
{|\vec{p}-\vec{m}|}
\left(1 - \frac{d}{R}\right)
\]

Used only when the mouse is within radius **R**.

---

## 7. Audio Reactivity (Optional)

Audio amplitude increases turbulence:

\[
reaction = 1 + 2 \cdot audioLevel
\]

\[
\vec{v} \leftarrow \vec{v} \cdot reaction
\]

Louder audio = more chaotic motion.

---

#  GPU Architecture

## 1. GPGPU Simulation

The simulation stores particle states in:
- `texturePosition` (RGBA32F)
- `textureVelocity` (RGBA32F)

Each **fragment** = 1 particle.

Each frame:
1. Velocity shader → computes new velocity  
2. Position shader → computes new positions  
3. Rendering shader → draws the particles  

This avoids CPU cost entirely.

---

## 2. Instanced Rendering

A single quad geometry is drawn **N times** using `InstancedBufferGeometry`.  
Each instance knows which particle it corresponds to through:


`reference = (u,v)` points to the correct pixel in the GPGPU texture.

---

## 3. Billboard Rendering

Particles are rendered as glowing quads that **face the camera**:

\[
offset = cameraRight \cdot u + cameraUp \cdot v
\]

Stretching based on velocity magnitude:

\[
stretch = \max(1,\ |\vec{v}| \cdot 2)
\]

---

## 4. Color Based on Energy

Color blends between two palettes depending on speed:

\[
t = smoothstep(0,5,|\vec{v}|)
\]

\[
color = mix(colorB, colorA, t)
\]

Slow = blue/soft  
Fast = orange/hot  

---

# Rendering Pipeline

1. **Scene render pass**
2. **UnrealBloomPass**
   - threshold  
   - strength  
   - radius  
3. Final composite to screen

This creates the glowing plasma effect.

---

# Shader Summary

## Velocity Shader
- Samples curl noise
- Updates velocity
- Applies mouse force
- Adds audio reactivity
- Applies decay

## Position Shader
- Adds velocity × delta
- Reduces life
- Respawns particle when dead

## Render Vertex Shader
- Reads position & velocity from textures
- Builds billboard quad
- Applies stretching
- Computes energy-dependent color

## Render Fragment Shader
- Draws circular glowing particle
- Applies soft falloff + transparency

---

# Running Locally

You must serve over a local server (due to module imports & mic permissions).

### Python
```bash
python3 -m http.server

Open:
```bash
http://localhost:8000

Node (recommended)

Use any dev server such as:

npx serve .

 Customization
Particle count

Edit:

SETTINGS.textureSize = 512; // or 1024 for 1M

Colors
SETTINGS.colorA = '#ff4d00';
SETTINGS.colorB = '#0066ff';

Turbulence
SETTINGS.curlStrength = 3.0;
SETTINGS.noiseScale = 1.5;

Particle size
SETTINGS.particleSize = 1.0;

 Key Equations Summary

Advection

𝑝
′
=
𝑝
+
𝑣
Δ
𝑡
p
′
=p+vΔt

Curl Noise

𝑐
𝑢
𝑟
𝑙
=
∇
×
𝐹
curl=∇×F

Velocity blending

𝑣
=
(
1
−
𝛼
)
𝑣
+
𝛼
⋅
𝑐
𝑢
𝑟
𝑙
v=(1−α)v+α⋅curl

Life decay

𝑙
𝑖
𝑓
𝑒
=
𝑙
𝑖
𝑓
𝑒
−
𝑑
𝑖
𝑒
𝑆
𝑝
𝑒
𝑒
𝑑
life=life−dieSpeed

Audio influence

𝑣
=
𝑣
(
1
+
2
 
𝑎
𝑢
𝑑
𝑖
𝑜
𝐿
𝑒
𝑣
𝑒
𝑙
)
v=v(1+2audioLevel)
 License

MIT License — free to use, modify, and extend.

 Credits

Built with:

Three.js

GPUComputationRenderer

GLSL

WebGL2
