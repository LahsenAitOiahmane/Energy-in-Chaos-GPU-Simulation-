# Energy in Chaos
**Interactive GPU Visualization**

*   **Author:** AIT OIHMANE Lahsen
*   **Tech Stack:** WebGL2, Three.js, GLSL, GPGPU
*   **License:** MIT

---

### Project Profile

A real-time, interactive visualization of turbulent flow fields, running entirely in the browser. This project demonstrates advanced use of GPGPU (General-Purpose computing on Graphics Processing Units) to simulate and render up to one million particles at a smooth 60 FPS. The system creates fluid-like, chaotic motion and dynamic energy visuals without relying on a traditional game engine or backend server.

### Key Features

*   **Massive Scale:** Efficiently simulates and renders up to 1,000,000 particles in real-time.
*   **GPU-Accelerated Physics:** All particle physics are calculated on the GPU using fragment shaders, freeing the CPU for other tasks.
*   **Divergence-Free Flow:** Employs a Curl Noise algorithm to generate realistic, incompressible fluid-like motion where particles form vortices instead of collapsing.
*   **Audio Reactivity:** The visualization can react to live microphone input, with particle energy and turbulence responding to sound levels.
*   **Dynamic Visuals:** Features a vibrant neon aesthetic achieved with HDR rendering, additive blending, and a bloom post-processing effect.
*   **Interactive Controls:** Users can rotate, pan, and zoom the camera. A mouse cursor can also repel particles, creating interactive disturbances.
*   **Highly Scalable:** Includes performance presets (Low, Medium, High, Ultra) to dynamically adjust the particle count for different hardware capabilities.

### Technical Implementation

*   **GPGPU Simulation:** Uses Three.js's `GPUComputationRenderer` to run simulation logic on the GPU. Particle state (position, velocity, life) is stored in floating-point textures.
*   **Ping-Pong Buffers:** A technique where the GPU reads from one texture and writes to a second texture each frame, then swaps them. This bypasses the limitation of not being able to read and write to the same texture simultaneously.
*   **Instanced Rendering:** Renders a million particles using a single draw call by drawing one plane geometry and "instancing" it for each particle, with a shader positioning each instance individually.
*   **Custom GLSL Shaders:**
    *   **Velocity Shader:** Calculates the forces on each particle using curl noise, audio input, and mouse interaction.
    *   **Position Shader:** Updates each particle's position based on its velocity and handles its lifecycle (respawning).
    *   **Render Shader:** Visually renders each particle as a stretched, glowing billboard.

---

### How the Main Script Works

The main script (`script.js`) orchestrates the entire application in two main phases: initialization and the animation loop.

#### 1. Initialization (`init()` function)

This function runs once at the start to set up the entire scene.

1.  **Setup Core Three.js:** It creates the fundamental components: the `Scene` (the world), the `Camera` (the viewpoint), and the `Renderer` (the canvas that draws the scene).
2.  **Initialize GPGPU (`initComputeRenderer`):**
    *   It creates a special `GPUComputationRenderer`.
    *   It defines two textures to hold particle data: one for `position` (x, y, z, life) and one for `velocity` (vx, vy, vz).
    *   It assigns the GLSL code (the "shaders") that will run on the GPU to update these textures every frame.
3.  **Create Particles (`initParticles`):**
    *   It creates a single, tiny plane geometry.
    *   It uses `InstancedBufferGeometry` to tell the GPU to draw this plane hundreds of thousands of times.
    *   It creates a special `reference` attribute that tells each instance which pixel in the GPGPU texture corresponds to its data.
4.  **Setup Post-Processing (`initPostProcessing`):** It creates an `EffectComposer` to manage post-processing effects, specifically adding an `UnrealBloomPass` to create the signature glow.
5.  **Create GUI & Listeners:** It builds the on-screen control panel (`lil-gui`) and sets up event listeners for window resizing and mouse movement.
6.  **Start the Loop:** Finally, it calls the `animate()` function for the first time, starting the continuous rendering cycle.

#### 2. The Animation Loop (`animate()` function)

This function runs continuously, typically 60 times per second, to create the illusion of motion.

1.  **Update Uniforms:** It gathers current data—like the elapsed time, mouse position, and audio levels—and sends this data to the GPU shaders.
2.  **Compute Simulation Step:** It calls `gpuCompute.compute()`. This command tells the GPU to execute the simulation shaders (Velocity and Position) for one frame. The GPU reads the particle data from the input textures, calculates the new state, and writes the results to the output textures.
3.  **Update Render Material:** It takes the newly updated position and velocity textures from the GPGPU renderer and passes them to the visual particle material.
4.  **Render the Scene:** It tells the `EffectComposer` to render the final image. This involves drawing all the instanced particles using their new positions and then applying the bloom effect.
5.  **Repeat:** It calls `requestAnimationFrame(animate)`, which is a browser API that schedules the `animate` function to be run again just before the next screen repaint.
