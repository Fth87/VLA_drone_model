# VLA Drone Simulator System

Sistem Vision Language Action (VLA) untuk kontrol drone berbasis AI. Terdiri dari 3 komponen: model inferensi Pi0, backend API Kaggle, dan frontend simulator 3D yang terhubung real-time.

## Arsitektur Sistem

![VLA Drone Architecture Diagram](/diagram.png)

---

## 📦 Komponen

### 1. Model (HuggingFace)

**Repository:** https://huggingface.co/izunx/drone-roblok-2500

- Base Model: pi0 (Vision Language Action)
- Fine-tuning: LoRA (efisien)
- Input: RGB image + text instruction
- Output: 4D action vector [Vx, Vy, Vz, Yaw]

### 2. Backend (Kaggle + Gradio)

**Repository:** https://github.com/Fth87/VLA_drone_backend_web_simulator-

- Runtime: Kaggle Notebooks (GPU T4)
- Framework: Gradio
- Public URL: `https://xxxxx.gradio.live`

**API Endpoint:**

```
POST /infer
Content-Type: multipart/form-data

Request:
  - image_input (required): RGB image file (224x224)
  - task_input (required): Text prompt ("terbang maju", "belok kanan")
  - state_input (optional): Drone state context

Response:
{
  "success": boolean,
  "first_action": [vx, vy, vz, yaw],    // range: -1..1
  "trajectory": [array] | null,
  "inference_time_ms": number,
  "error": null | "error message"
}
```

### 3. Frontend (Netlify)

**Repository:** https://github.com/Fth87/VLA_drone_web_simulator

- Live URL: https://drone-vla-simulator.netlify.app/
- Framework: TanStack Start + Three.js + React Three Fiber
- Auth: Supabase

---

## 🚀 Setup & Deploy

### Step 1: Train Model (Kaggle Notebook)

1. Open `Pi0Training.ipynb` di Kaggle
2. Attach dataset: `sjankaczar/dronepivla`
3. Run all cells
4. Config (STEP 5):
   ```
   NUM_STEPS: 3000          # train iterations
   BATCH_SIZE: 1            # per iter (safe for T4)
   SAVE_INTERVAL: 500       # checkpoint frequency
   LEARNING_METHOD: LoRA    # efficient method
   ```
5. Monitor progress (STEP 10)
6. Download checkpoint: `model.safetensors` (STEP 11)

**Output:** `/kaggle/working/checkpoints/...`

### Step 2: Run Backend (Kaggle Notebook)

1. Open `backend.ipynb` di Kaggle
2. Ensure GPU enabled
3. Run all cells
4. Wait for: `"Running on public URL: https://xxxxx.gradio.live"`
5. **Copy that URL** → save for frontend config

⚠️ **Note:** Kaggle sessions have time limit. For 24/7, migrate to dedicated GPU server.

### Step 3: Setup Frontend

```bash
git clone https://github.com/Fth87/VLA_drone_web_simulator.git
cd VLA_drone_web_simulator
cp .env.example .env
```

Edit `.env`:

```bash
VITE_VLA_API_URL=https://xxxxx.gradio.live   # from Step 2
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-public-key
```

Run:

```bash
pnpm install
pnpm dev          # http://localhost:3000
```

**Test Login:**

```
Email:    admin@gmail.com
Password: adminadmin
```

---

## 🎮 Controls

**Keyboard:**
| Key | Action |
|-----|--------|
| `W` / `S` | Forward/Backward (vz) |
| `Q` / `E` | Up/Down (vy) |
| `A` / `D` | Rotate left/right (yaw) |
| `←` / `→` | Strafe left/right (vx) |
| `1` / `3` / `4` | FPV / 3rd-person / fixed camera |

**Manual Control (UI Sliders):**

- Vx (lateral): -1 to 1
- Vy (vertical): -1 to 1
- Vz (forward): -1 to 1
- Yaw (rotation): -1 to 1

**VLA Inference (Autonomous):**

1. Type prompt in text field (e.g., "go to red box")
2. Click **Start Inference**
3. Drone executes predictions real-time
4. View telemetry in HUD

---

## ⚙️ How It Works

**Real-time Inference Loop (Every ~100ms):**

```
Frontend:
1. Capture canvas screenshot (224x224 JPEG)
2. Build FormData {image_input, task_input}
3. POST to ${VITE_VLA_API_URL}/infer
   ↓
Backend (Kaggle):
4. Receive multipart form
5. Load & preprocess image
6. Tokenize instruction
7. Run policy forward pass
8. Extract [vx, vy, vz, yaw]
9. Return JSON response
   ↓
Frontend:
10. Parse first_action
11. Apply ACTION_GAIN multiplier
12. Update drone physics
13. Re-render + update telemetry HUD
14. (loop continues)
```

---

## 🛠️ Configuration

**File:** `src/features/drone-sim/constants.ts`

```ts
// Control sensitivity
export const ACTION_GAIN = {
  vx: 1.8, // Lateral (↔)
  vy: 1.4, // Vertical (↑↓)
  vz: 2.1, // Forward (←→)
  yaw: 0.9, // Rotation (⟲)
};

// Flight boundaries (units)
export const DRONE_BOUNDS = {
  x: 28, // Lateral limit
  yMin: 0.35, // Floor
  yMax: 12, // Ceiling
  z: 28, // Depth
};

// Map/scene
export const MAP_FOOTPRINT_SIZE = 128;
export const MAP_MODEL_URL = '/maps/maps_roblox.glb';
export const DRONE_MODEL_URL = '/drone%20model/drone_model.glb';

// VLA tuning
export const VLA_INFERENCE_INTERVAL_MS = 100;
export const VLA_IMAGE_SIZE = 224;
export const VLA_REQUEST_TIMEOUT_MS = 10_000;
```

### Custom Maps/Drone Models

**Replace Map:**

1. Export dari Roblox → Save as `.glb`
2. Copy ke `public/maps/your-map.glb`
3. Update `constants.ts`: `MAP_MODEL_URL = '/maps/your-map.glb'`

**Replace Drone:**

1. Export dari Roblox → Save as `.glb`
2. Copy ke `public/drone model/your-drone.glb`
3. Update: `DRONE_MODEL_URL = '/drone%20model/your-drone.glb'` (%20 = space)

---

## 📊 Data Collection

**Flow:**

```
Roblox Studio (user demonstrates)
    ↓
Script captures: image, position, velocity, action
    ↓
POST to Data Collector API
    ↓
Store: {frame: image, action: [vx, vy, vz, yaw]}
    ↓
Pipeline Processing → LeRobot dataset format
    ↓
Use for training in Kaggle
```

**Components:**

- Repository: https://github.com/Fth87/VLA_drone_data_collector
- Server: Python API (listens to Roblox requests)
- Output: HuggingFace LeRobot dataset

---

## 🐛 Troubleshooting

| Problem                           | Solution                              |
| --------------------------------- | ------------------------------------- |
| **Training: Memory error**        | Reduce `BATCH_SIZE` or `NUM_STEPS`    |
| **Training: Dataset not found**   | Attach `dronepivla` dataset in Kaggle |
| **Training: Import errors**       | Restart kernel after STEP 2           |
| **Backend: Timeout >1 hour**      | Kaggle session expired, restart       |
| **Backend: /infer slow**          | Normal ~1-2 sec. Check GPU load       |
| **Frontend: Can't reach backend** | Verify `VITE_VLA_API_URL` is correct  |
| **Frontend: Lag/stuttering**      | Reduce `VLA_INFERENCE_INTERVAL_MS`    |
| **Frontend: .glb won't load**     | Ensure valid export from Roblox       |
| **Frontend: Drone crashes**       | Increase `DRONE_BOUNDS.yMin`          |
| **Inference slow**                | Check network latency to backend      |

---

## 📝 Complete Workflow

```
1️⃣  Roblox Studio
    └─> User demonstrates drone control
    └─> Captured data → Collector API

2️⃣  Kaggle: Training
    └─> Pi0Training.ipynb
    └─> Output: model.safetensors

3️⃣  Kaggle: Backend
    └─> backend.ipynb
    └─> Expose: https://xxxxx.gradio.live

4️⃣  Netlify/Local: Frontend
    └─> Connect to backend
    └─> Real-time drone simulation
    └─> User types prompts

5️⃣  Loop: Collect New Data
    └─> Fine-tune model
    └─> Deploy new version
```

---

**Last Updated:** April 2026 | **Status:** Production Ready (Dev Phase) | **Team:** AI Robotics Lab
