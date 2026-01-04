<template>
  <div class="mercury-engine">
    <canvas ref="canvas" class="webgl-canvas"></canvas>
    <div class="ui-layer">
      <div class="ui-content">
        <div class="badge-wrapper">
          <span class="system-badge">Spatial Interface v2.0</span>
        </div>
        <h1 class="main-title">
          Mercury
          <span class="liquid-text">Visions</span>
        </h1>
        <p class="description">
          A purely procedural world of liquid metal and high-contrast light. 
          Respond to the flow of physical simulations in real-time.
        </p>
        <div class="cta-row">
          <button class="btn btn-primary">Start Vision</button>
          <button class="btn btn-secondary">Specifications</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'

const canvas = ref(null)
let renderer, scene, camera, mesh, animationId
const mouse = new THREE.Vector2(0.5, 0.5)
const targetMouse = new THREE.Vector2(0.5, 0.5)

// --- SHADERS ---

const vertexShader = `
  varying vec2 vUv;
  void main() {
    vUv = uv;
    gl_Position = vec4(position, 1.0);
  }
`

const fragmentShader = `
  precision highp float;
  uniform float uTime;
  uniform vec2 uResolution;
  uniform vec2 uMouse;
  varying vec2 vUv;

  #define MAX_STEPS 90
  #define MAX_DIST 40.0
  #define SURF_DIST 0.001

  // Smooth Union for fluid merging
  float smin(float a, float b, float k) {
    float h = clamp(0.5 + 0.5 * (b - a) / k, 0.0, 1.0);
    return mix(b, a, h) - k * h * (1.0 - h);
  }

  // SDF Scene - High-Density Mercury Soup
  float getSceneSDF(vec3 p) {
    float t = uTime * 0.12;
    vec2 mPos = (uMouse - 0.5) * 6.5;
    
    // Attractor
    float d = length(p - vec3(mPos.x, mPos.y, 1.2)) - 0.9;
    
    // Large Fluid Bodies (8 bodies) - Squash & Stretch
    for(int i = 0; i < 8; i++) {
        float fi = float(i);
        vec3 pos = vec3(
            sin(t * 0.7 + fi * 1.5) * 3.8,
            cos(t * 0.5 + fi * 2.2) * 2.2,
            sin(t * 0.9 + fi * 0.8) * 1.0
        );
        vec3 p_str = p - pos;
        p_str.y *= 0.8 + 0.3 * sin(t + fi); // Organic pulse
        float r = 0.5 + 0.3 * sin(t * 0.8 + fi);
        d = smin(d, length(p_str) - r, 1.1); 
    }
    
    // Micro Beads (16 beads for density)
    for(int j = 0; j < 16; j++) {
        float fj = float(j);
        vec3 pos = vec3(
            sin(t * 1.3 + fj * 4.1) * 5.2,
            cos(t * 1.0 + fj * 2.8) * 3.6,
            sin(t * 1.6 + fj * 0.7) * 0.8
        );
        float r = 0.08 + 0.2 * abs(cos(t * 1.8 + fj * 1.2));
        d = smin(d, length(p - pos) - r, 0.45); 
    }
    
    d += sin(p.x * 2.5 + t) * sin(p.y * 2.5 + t) * 0.06;
    return d;
  }

  // Normal via gradient
  vec3 getNormal(vec3 p) {
    vec2 e = vec2(0.001, 0.0);
    vec3 n = vec3(
        getSceneSDF(p + e.xyy) - getSceneSDF(p - e.xyy),
        getSceneSDF(p + e.yxy) - getSceneSDF(p - e.yxy),
        getSceneSDF(p + e.yyx) - getSceneSDF(p - e.yyx)
    );
    return normalize(n);
  }

  // Studio Lighting Environment (Ultra-Chrome Studio)
  vec3 getEnvironment(vec3 r) {
    // Neural High-Key Base
    vec3 col = vec3(1.0);
    
    // Top Studio Softbox (High Energy)
    float overhead = smoothstep(-0.5, 0.5, r.y);
    col = mix(vec3(0.85, 0.9, 0.96), vec3(1.0), overhead);
    
    // SHARP Side Light Bars (The "Chrome" Definition)
    // We use tight dot products to simulate actual studio lamps
    float bar1 = pow(max(0.0, dot(r, normalize(vec3(1.5, 0.2, -0.8)))), 24.0);
    float bar2 = pow(max(0.0, dot(r, normalize(vec3(-1.2, 0.1, -0.4)))), 32.0);
    col += vec3(1.8) * bar1 * overhead;
    col += vec3(1.2) * bar2 * (1.0 - overhead);
    
    // Deep Studio Occluders (Black Panels)
    // This creates the "Mirror" contrast
    float panelA = smoothstep(0.4, 0.9, abs(r.x));
    float panelB = smoothstep(0.2, 0.7, abs(r.z));
    col = mix(col, vec3(0.0), panelA * 0.85 * (1.0 - overhead));
    col = mix(col, vec3(0.01), panelB * 0.65);
    
    // Crisp Studio Strips (Secondary Gloss)
    float strips = pow(abs(cos(r.x * 2.5 + r.z * 1.5 + uTime * 0.05)), 60.0);
    col = mix(col, vec3(1.5), strips * 0.4 * overhead);
    
    return col;
  }

  void main() {
    vec2 uv = (gl_FragCoord.xy - 0.5 * uResolution.xy) / uResolution.y;
    
    // High-Aperture Macro Camera
    vec3 ro = vec3(0.0, 0.0, -8.2);
    vec3 rd = normalize(vec3(uv, 3.2)); 
    
    float dTotal = 0.0;
    for(int i = 0; i < MAX_STEPS; i++) {
        vec3 p = ro + rd * dTotal;
        float dS = getSceneSDF(p);
        dTotal += dS;
        if(dTotal > MAX_DIST || abs(dS) < SURF_DIST) break;
    }
    
    // Pure White Background
    vec3 col = vec3(1.0);
    
    if(dTotal < MAX_DIST) {
        vec3 p = ro + rd * dTotal;
        vec3 n = getNormal(p);
        vec3 v = -rd;
        vec3 r = reflect(rd, n);
        
        // --- ULTRA-CHROME MIRROR BRDF ---
        vec3 reflection = getEnvironment(r);
        
        // Polished Silver Base
        col = reflection * vec3(0.98, 0.99, 1.0);
        
        // METALLIC GLOSS ENGINE
        vec3 lightPos = normalize(vec3((uMouse.x - 0.5) * 15.0, (uMouse.y - 0.5) * 15.0 + 5.0, -3.5));
        
        // 1. Terminal Specular Glint (Sharp Peak)
        float specPeak = pow(max(0.0, dot(reflect(-lightPos, n), v)), 4000.0);
        col += vec3(3.5) * specPeak;
        
        // 2. Anisotropic Rim Glow
        float fres = pow(1.0 - max(0.0, dot(n, v)), 6.0);
        col = mix(col, vec3(1.0), fres * 0.75); // Extreme edge brightness
        
        // 3. Pearlescent Interference (High-end metallic detail)
        col += fres * vec3(0.6, 0.8, 1.2) * 0.6;
        
        // 4. Studio Specular Bloom
        float specSoft = pow(max(0.0, dot(reflect(-lightPos, n), v)), 64.0);
        col += vec3(0.4, 0.6, 1.0) * specSoft * 0.35;
        
        // Ambient Occlusion / Mass Definition
        float curve = 1.0 - abs(dot(n, vec3(0,0,1)));
        col *= mix(1.0, 0.7, pow(curve, 3.5));
    }
    
    // Final Toning (Contrast & Dynamic Range)
    col = pow(col, vec3(0.85)); // Deepen chrome, pop highlights
    col = clamp(col, 0.0, 1.0);
    
    gl_FragColor = vec4(col, 1.0);
  }
`

const init = () => {
  renderer = new THREE.WebGLRenderer({ canvas: canvas.value, antialias: true, powerPreference: 'high-performance' })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))

  scene = new THREE.Scene()
  camera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1)

  const material = new THREE.ShaderMaterial({
    uniforms: {
      uTime: { value: 0 },
      uResolution: { value: new THREE.Vector2(window.innerWidth, window.innerHeight) },
      uMouse: { value: new THREE.Vector2(0.5, 0.5) }
    },
    vertexShader, fragmentShader
  })

  mesh = new THREE.Mesh(new THREE.PlaneGeometry(2, 2), material)
  scene.add(mesh)

  const tick = () => {
    animationId = requestAnimationFrame(tick)
    mouse.x += (targetMouse.x - mouse.x) * 0.08
    mouse.y += (targetMouse.y - mouse.y) * 0.08
    material.uniforms.uTime.value = performance.now() * 0.001
    material.uniforms.uMouse.value.set(mouse.x, mouse.y)
    material.uniforms.uResolution.value.set(window.innerWidth, window.innerHeight)
    renderer.render(scene, camera)
  }
  tick()
}

onMounted(() => {
  init()
  window.addEventListener('resize', () => {
    renderer.setSize(window.innerWidth, window.innerHeight)
    mesh.material.uniforms.uResolution.value.set(window.innerWidth, window.innerHeight)
  })
  window.addEventListener('mousemove', (e) => {
    targetMouse.x = e.clientX / window.innerWidth
    targetMouse.y = 1.0 - (e.clientY / window.innerHeight)
  })
})

onBeforeUnmount(() => {
  cancelAnimationFrame(animationId)
  renderer.dispose()
})
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;600;700;800&display=swap');

.mercury-engine {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background: #fff;
  font-family: 'Plus Jakarta Sans', sans-serif;
}

.webgl-canvas {
  position: absolute;
  top: 0; left: 0;
  width: 100%; height: 100%;
  z-index: 1;
}

.ui-layer {
  position: relative;
  z-index: 2;
  width: 100%; height: 100%;
  display: flex;
  align-items: center; justify-content: center;
  pointer-events: none;
}

.ui-content {
  max-width: 900px;
  text-align: center;
  padding: 0 40px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1.8rem;
}

.system-badge {
  display: inline-block;
  padding: 0.7rem 1.8rem;
  background: rgba(255, 255, 255, 0.4);
  backdrop-filter: blur(20px);
  border-radius: 200px;
  border: 1px solid rgba(0,0,0,0.06);
  font-size: 0.72rem;
  font-weight: 700;
  letter-spacing: 0.2em;
  text-transform: uppercase;
  color: rgba(0,0,0,0.45);
  pointer-events: auto;
}

.main-title {
  font-size: clamp(3.8rem, 14vw, 8.5rem);
  font-weight: 800;
  line-height: 0.82;
  letter-spacing: -0.05em;
  color: #000;
  margin: 0;
  pointer-events: auto;
}

.liquid-text {
  display: block;
  background: linear-gradient(180deg, #111 0%, #777 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.description {
  font-size: clamp(1.15rem, 2.8vw, 1.45rem);
  color: rgba(0,0,0,0.48);
  font-weight: 400;
  line-height: 1.55;
  max-width: 620px;
  pointer-events: auto;
}

.cta-row {
  margin-top: 2rem;
  display: flex;
  gap: 1.6rem;
  pointer-events: auto;
}

.btn {
  padding: 1.4rem 3.4rem;
  border-radius: 200px;
  font-size: 1.1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.6s cubic-bezier(0.16, 1, 0.3, 1);
  border: none;
}

.btn-primary {
  background: #000;
  color: #fff;
  box-shadow: 0 12px 45px rgba(0,0,0,0.12);
}

.btn-primary:hover {
  transform: translateY(-8px) scale(1.04);
  box-shadow: 0 24px 60px rgba(0,0,0,0.24);
}

.btn-secondary {
  background: rgba(255, 255, 255, 0.6);
  backdrop-filter: blur(25px);
  color: #000;
  border: 1px solid rgba(0,0,0,0.12);
}

.btn-secondary:hover {
  transform: translateY(-8px);
  background: rgba(255, 255, 255, 0.95);
}

@media (max-width: 768px) {
  .cta-row { flex-direction: column; width: 100%; max-width: 320px; }
  .btn { width: 100%; }
}
</style>
