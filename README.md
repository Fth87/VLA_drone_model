# VLA Drone Simulator System

Sistem Vision Language Action (VLA) untuk kontrol drone drone berbasis AI. Terdiri dari tiga komponen utama: model inferensi, backend API, dan frontend simulator 3D yang terhubung real-time.

## Arsitektur Sistem

![VLA Drone Architecture Diagram](/diagram.png)

## Komponen Sistem

### 1. Model Inferensi (HuggingFace)

Model VLA Pi0 yang sudah fine-tuned untuk kontrol drone berbasis vision dan language.

**Spesifikasi:**

- Repository: https://huggingface.co/izunx/drone-roblok-2500
- Base Model: pi0 (Vision Language Action pre-trained)
- Fine-tuning: Custom dataset dari Roblox + real drone data
- Input: RGB image (dari drone camera) + text instruction ("terbang maju", "belok kanan", dll)
- Output: 4D action vector (Vx, Vy, Vz, Yaw) untuk drone control
- Training Method: LoRA untuk efficiency

### 2. Backend API (Kaggle + Gradio)

Backend inference engine yang berjalan di Kaggle Notebook dengan GPU untuk serve model Pi0 secara real-time.

**Spesifikasi:**

- Repository: https://github.com/Fth87/VLA_drone_backend_web_simulator-
- Runtime: Kaggle Notebooks dengan GPU aktif
- Framework: Gradio (auto-expose endpoints -> public HTTPS URL)
- Public URL Format: `https://xxxxx.gradio.live`
- Model Loading: Kebijakan (policy) dibuat sekali, reused untuk setiap inference

**API Endpoints:**

**1. POST /infer** (Main inference endpoint)

```
Content-Type: multipart/form-data

Fields:
- image_input (required): Image file dari drone camera
- task_input (required): Text instruction/prompt ("terbang maju", "belok kanan")
- state_input (optional): List string untuk drone state

Response:
{
  "success": boolean,
  "first_action": [Vx, Vy, Vz, Yaw],  // 4D action vector
  "trajectory": [...],                  // predicted trajectory
  "inference_time_ms": number,          // latency for monitoring
  "error": string or null               // error message if failed
}
```

**2. GET/POST /health** (Liveness check)
Simple endpoint untuk check apakah backend aktif

**Backend Processing Flow:**

1. Receive multipart form data (image + task prompt)
2. Load image file
3. Preprocess: normalize, resize ke model input size
4. Tokenize task instruction dalam context prompt
5. Run policy inference (single forward pass)
6. Extract 4D action vector + trajectory prediction
7. Return JSON response dengan timing info

### 3. Frontend Simulator (Netlify)

Web-based 3D simulator yang memvisualisasi drone environment dan menampilkan hasil inference real-time.

**Spesifikasi:**

- Live URL: https://drone-vla-simulator.netlify.app/
- Repository: https://github.com/Fth87/VLA_drone_web_simulator
- Framework: TanStack Start
- 3D Rendering: Three.js + React Three Fiber
- Authentication: Supabase (user management)
- 3D Assets: Custom .glb import (Roblox Studio format)

**Test Login (Frontend):**
```
Email:    admin@gmail.com
Password: adminadmin
```

**Environment Variables:**

```bash
VITE_VLA_API_URL=https://xxxxx.gradio.live    # From Kaggle backend
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-public-key
```

**Run Locally:**

```bash
git clone https://github.com/Fth87/VLA_drone_web_simulator.git
cd VLA_drone_web_simulator
cp .env.example .env          # Edit dengan VITE_VLA_API_URL
pnpm install
pnpm dev                      # http://localhost:3000
```



**Controls:**

| Key/Control     | Action                                 |
| --------------- | -------------------------------------- |
| `W` / `S`       | Forward/Backward (vz)                  |
| `Q` / `E`       | Up/Down (vy)                           |
| `A` / `D`       | Rotate left/right (yaw)                |
| `←` / `->`       | Strafe left/right (vx)                 |
| `1` / `3` / `4` | FPV / 3rd-person / fixed corner camera |
| `click canvas`  | Focus for keyboard input               |

**Manual Control (via UI Sliders):**

- Vx (lateral): -1 to 1 (left to right)
- Vy (vertical): -1 to 1 (down to up)
- Vz (forward): -1 to 1 (back to forward)
- Yaw (rotation): -1 to 1 (counter-clockwise to clockwise)

**VLA Inference (Natural Language):**

1. Type prompt di input field (e.g., "go to red box")
2. Click **Start Inference**
3. Drone executes VLA predictions in real-time
4. View telemetry di HUD (position, heading, inference timing)
5. Download frame payload jika perlu

**Configuration (src/features/drone-sim/constants.ts):**

```ts
// Movement response tuning
export const ACTION_GAIN = {
  vx: 1.8, // Lateral speed sensitivity
  vy: 1.4, // Vertical speed sensitivity
  vz: 2.1, // Forward speed sensitivity
  yaw: 0.9, // Rotation sensitivity
};

// Flight boundaries (units)
export const DRONE_BOUNDS = {
  x: 28, // Lateral limit
  yMin: 0.35, // Floor height
  yMax: 12, // Ceiling height
  z: 28, // Depth limit
};

// Map size
export const MAP_FOOTPRINT_SIZE = 128; // Width/depth units

// VLA tuning
export const VLA_INFERENCE_INTERVAL_MS = 100; // Loop frequency
export const VLA_IMAGE_SIZE = 224; // Capture resolution
export const VLA_REQUEST_TIMEOUT_MS = 10_000; // 10 sec timeout
```

**Custom Assets (Roblox Studio):**

_Replace Map:_

1. Export dari Roblox -> Save as `.glb` format
2. Copy ke `public/maps/your-map.glb`
3. Update `constants.ts`: `export const MAP_MODEL_URL = '/maps/your-map.glb'`
4. Auto-scales ke `MAP_FOOTPRINT_SIZE`

_Replace Drone Model:_

1. Export dari Roblox -> Save as `.glb`
2. Copy ke `public/drone model/your-drone.glb`
3. Update path: `export const DRONE_MODEL_URL = '/drone%20model/your-drone.glb'` (note: %20 = space)

**Fitur Utama:**

- Real-time 3D visualization drone + custom map
- Multi-camera modes (FPV, 3rd-person, fixed view)
- Hybrid control: keyboard + sliders + VLA inference
- Live telemetry & HUD display
- Payload frame capture & download
- Performance monitoring (inference timing)

**Frontend Inference Loop:**

```
Setiap frame (~60fps):
1. Capture screenshot dari Three.js canvas (224x224)
2. Create FormData
   - image_input: JPEG binary
   - task_input: user prompt text
3. POST /infer ke backend (multipart/form-data)
4. Parse response -> extract first_action [vx, vy, vz, yaw]
5. Apply ACTION_GAIN multiplier
6. Update drone position/rotation via physics
7. Re-render scene + update HUD telemetry
```

## Frontend-Backend Integration

**Real-time Inference Workflow:**

1. **Frontend :** Render 3D scene
2. **Every frame:**
   - Capture canvas -> 224x224 JPEG
   - Build FormData: {image_input, task_input}
   - POST to `${VITE_VLA_API_URL}/infer`
3. **Backend (Kaggle Notebook):**
   - Receive multipart form -> load image
   - Preprocess & tokenize instruction
   - Forward through policy -> extract [vx, vy, vz, yaw]
   - Return JSON: {success, first_action, trajectory, inference_time_ms}
4. **Frontend :**
   - Parse first_action -> apply ACTION_GAIN
   - Update drone position using action vector
   - Draw trajectory visualization
   - Update HUD with timing info
5. **Loop:** Continues every 100ms (tunable) while simulation active

## Data Collection Pipeline (Roblox Studio + Data Collector)

Dataset training dikumpulkan secara otomatis dari user interactions di Roblox Studio menggunakan API-based data collector.

**Komponen:**

- Repository: https://github.com/Fth87/VLA_drone_data_collector
- Collector Server: Python API yang listen ke Roblox requests
- Roblox Integration: Built-in script automatically log setiap action

**Data Collection Flow:**

```
Roblox Studio (User plays + guides drone)
    ↓
Roblox Script -> Capture state (image, position, velocity)
    ↓
HTTP POST to Data Collector API
    ↓
Data stored as {frame: image, action: [Vx, Vy, Vz, Yaw]}
    ↓
Accumulate -> Process -> LeRobot dataset format
    ↓
Ready for Kaggle training
```

**Data Collected:**

- **Image:** Drone camera feed (RGB dari Roblox viewport)
- **Action:** 4D control vector (Vx, Vy, Vz velocities + Yaw rotation)
- **Metadata:** Timestamp, episode ID, instruction text
- **Output:** HuggingFace LeRobot dataset (compatible dengan training notebook)

## Running the Backend

**Setup & Launch:**

1. Buka notebook di Kaggle: `backend.ipynb`
2. Pastikan GPU aktif
3. Klik "Run All" -> tunggu dependency install + model load
4. Backend akan publish URL publik format `https://xxxxx.gradio.live`
5. Copy URL tersebut -> paste ke frontend environment `VITE_VLA_API_URL`
6. Frontend siap call backend untuk inference real-time

**Backend Notebook Steps:**

- Install & import dependencies
- Apply runtime patch untuk kompatibilitas OpenPI
- Load model checkpoint dari HuggingFace
- Create policy object (reused untuk setiap inference)
- Expose Gradio endpoints (/infer, /health)
- Listen untuk frontend requests

**Important:** Backend session di Kaggle ada time limit. Untuk production 24/7, perlu migrate ke dedicated GPU server.

## Training Model (Kaggle Notebook)

Pipeline fine-tuning otomatis Pi0 model menggunakan dataset custom. 12 steps menangani setup, training, monitoring, dan checkpoint management.

**Prasyarat:** Kaggle + GPU T4+ (15GB VRAM) + dataset `dronepivla`

**Quick Start:**

1. Upload notebook -> Attach dataset `sjankaczar/dronepivla`
2. STEP 1-2: Setup & install (restart kernel setelah ini)
3. STEP 3-7: Data prep & model setup
4. STEP 9: Run training (akan berjalan sesuai NUM_STEPS)
5. STEP 10: Monitor progress repeat berkali-kali
6. STEP 11: Download checkpoint (model.safetensors)

**Config (STEP 5):**

```
NUM_STEPS: 3000        # training iterations
BATCH_SIZE: 1          # per iteration (aman untuk T4)
SAVE_INTERVAL: 500     # checkpoint frequency
LEARNING_METHOD: LoRA  # efficient fine-tuning
```

**Output:** Checkpoint di `/kaggle/working/checkpoints/...` (format: safetensors)

## Troubleshooting & Production Notes

**Training Issues:**

- Memory error -> Turunkan `BATCH_SIZE` atau reduce `NUM_STEPS`
- Dataset not found -> Pastikan dataset `dronepivla` ter-attach di Kaggle
- Import errors -> Restart kernel setelah step 2 installation
- Training lambat -> Normal for T4, upgrade GPU untuk faster iteration

**Backend/API Issues:**

- Timeout > 1 jam -> Kaggle session limit, restart backend
- /infer endpoint slow -> Inference ~1-2 sec normal, check Kaggle GPU load

**Frontend Issues:**

- Not connecting to backend -> Verify backend URL di frontend config
- Simulator lag -> Reduce frame update frequency di frontend config
- .glb import fail -> Pastikan file format valid dari Roblox Studio export

**Architecture Notes:**

- Backend: Kaggle session tidak cocok untuk 24/7 production. Untuk demo/dev OK. Butuh migrate ke dedicated GPU server untuk production use.
- Model: Checkpoint format `safetensors` bisa langsung di-push ke HuggingFace Hub untuk versioning.
- Data: System continuously collect dari Roblox -> enable iterative fine-tuning cycles dengan data baru.
- Frontend: Can import custom .glb maps dari Roblox Studio untuk environment variety.

## Struktur File

```
/kaggle/working/
├── openpi/               # OpenPI repository source
├── .cache/
│   └── huggingface/
│       └── lerobot/      # Dataset cache directory
├── assets/               # Normalization stats & configs
└── checkpoints/          # Training output directory
    └── pi0_drone_lite/
        └── drone_gimbal_v1/
            └── LATEST/   # Latest checkpoint
                ├── model.safetensors
                ├── optimizer.pt
                └── config.json
```

## Workflow Lengkap

```
1. Data Collection (Roblox Studio)
   └─> VLA Data Collector API
       └─> Dataset accumulation

2. Model Training (Kaggle Notebook)
   └─> Dataset processing
       └─> Pi0 fine-tuning dengan LoRA
           └─> Checkpoint saved

3. Model Deployment (HuggingFace)
   └─> Upload checkpoint
       └─> Inference ready

4. Frontend Simulation (Netlify)
   └─> User instruction input
       └─> Real-time API calls ke backend
           └─> Visual feedback 3D
```

---

**Last Updated**: April 2026
**Status**: Production Ready (Demo/Development phase)
**Team**: AI Robotics Lab
