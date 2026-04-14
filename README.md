# VLA Drone Simulator System

Sistem Vision Language Action (VLA) untuk kontrol drone berbasis AI. Sistem terdiri dari 3 komponen utama: model inferensi, backend API, dan frontend simulator 3D yang terhubung real-time.

## Arsitektur Sistem

![VLA Drone Architecture Diagram](/diagram.png)

## Ringkasan

### 1. Model Inferensi

Model VLA Pi0 yang sudah fine-tuned untuk kontrol drone berbasis vision dan language.

- Repository: https://huggingface.co/izunx/drone-roblok-2500
- Base Model: pi0 (Vision Language Action pre-trained)
- Fine-tuning: Custom dataset dari Roblox + real drone data
- Input: RGB image (dari drone camera) + text instruction
- Output: 4D action vector `[vx, vy, vz, yaw]`
- Training Method: LoRA

### 2. Backend

Backend inference engine yang berjalan di Kaggle Notebook dengan GPU untuk serve model Pi0 secara real-time.

- Repository: https://github.com/Fth87/VLA_drone_backend_web_simulator-
- Runtime: Kaggle Notebooks dengan GPU aktif
- Framework: Gradio (auto-expose endpoint ke public HTTPS URL)
- Public URL format: `https://xxxxx.gradio.live`
- Model loading: policy dibuat sekali, dipakai ulang untuk setiap inference

### 3. Frontend Simulator 

Web-based 3D simulator untuk visualisasi environment drone dan hasil inference real-time.

- Live URL: https://drone-vla-simulator.netlify.app/
- Repository: https://github.com/Fth87/VLA_drone_web_simulator
- Framework: TanStack Start + Three.js + React Three Fiber
- Authentication: Supabase
- 3D assets: custom `.glb` import (dari Roblox Studio)

## End-to-End Workflow

1. Data dikumpulkan dari Roblox Studio melalui data collector.
2. Model dilatih/fine-tune di Kaggle (`Pi0Training.ipynb`).
3. Backend inference dijalankan di Kaggle (`backend.ipynb`) dan menghasilkan URL publik Gradio.
4. Frontend terhubung ke backend via `VITE_VLA_API_URL` untuk inferensi real-time.
5. Data baru bisa dipakai untuk retraining iteratif.

## Setup 

### Step 1: Train Model (Kaggle Notebook)

1. Open `Pi0Training.ipynb` di Kaggle.
2. Attach dataset `sjankaczar/dronepivla`.
3. Run semua cell.
4. Monitor training progress.
5. Download checkpoint `model.safetensors`.

Output checkpoint berada di `/kaggle/working/checkpoints/...`.

### Step 2: Run Backend (Kaggle Notebook)

1. Open `backend.ipynb` di Kaggle.
2. Pastikan GPU aktif.
3. Run semua cell.
4. Tunggu output URL publik, contoh: `https://xxxxx.gradio.live`.
5. Simpan URL ini untuk konfigurasi frontend.

### Step 3: Setup Frontend

```bash
git clone https://github.com/Fth87/VLA_drone_web_simulator.git
cd VLA_drone_web_simulator
cp .env.example .env
```

Isi `.env`:

```bash
VITE_VLA_API_URL=https://xxxxx.gradio.live
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-public-key
```

Jalankan frontend:

```bash
pnpm install
pnpm dev
```

Akses di `http://localhost:3000`.

Test login frontend:

```text
Email: admin@gmail.com
Password: adminadmin
```

## Backend API Specification

### Endpoint

`POST /infer`

Request content type: `multipart/form-data`

Request fields:
- `image_input` (required): image file dari camera/frame
- `task_input` (required): text instruction/prompt
- `state_input` (optional): state context drone

Contoh response:

```json
{
  "success": true,
  "first_action": [0.2, 0.1, 0.8, -0.3],
  "trajectory": null,
  "inference_time_ms": 245,
  "error": null
}
```

Format `first_action`: `[vx, vy, vz, yaw]` dengan rentang `-1..1`.

### Health Check

`GET /health` atau `POST /health` untuk cek backend aktif.

### Backend Processing Flow

1. Receive multipart form data (`image_input`, `task_input`, `state_input`).
2. Load image file.
3. Preprocess (normalize, resize ke input model).
4. Tokenize task instruction.
5. Run policy inference.
6. Extract action vector + trajectory.
7. Return JSON response + timing info.

## Frontend Usage

### Controls

| Key/Control | Action |
| --- | --- |
| `W` / `S` | Forward/Backward (vz) |
| `Q` / `E` | Up/Down (vy) |
| `A` / `D` | Rotate left/right (yaw) |
| `←` / `→` | Strafe left/right (vx) |
| `1` / `3` / `4` | FPV / 3rd-person / fixed corner camera |
| Click canvas | Focus keyboard input |

### Manual Control (UI Sliders)

- Vx (lateral): -1 to 1
- Vy (vertical): -1 to 1
- Vz (forward): -1 to 1
- Yaw (rotation): -1 to 1

### VLA Inference

1. Isi prompt pada input (contoh: "look at the red box").
2. Klik **Start Inference**.
3. Drone menjalankan prediksi VLA secara real-time.
4. Monitor telemetry di HUD.

### Frontend Inference Loop

1. Capture screenshot canvas (`224x224`, JPEG).
2. Buat `FormData` (`image_input`, `task_input`).
3. `POST` ke `${VITE_VLA_API_URL}/infer`.
4. Parse `first_action` dari response.
5. Apply `ACTION_GAIN` multiplier.
6. Update physics drone dan render ulang scene.


## Data Collection Pipeline

Dataset training dikumpulkan otomatis di Roblox Studio menggunakan data collector API.

Flow:

```text
Roblox Studio (user bermain dan mengarahkan drone)
  -> Roblox script capture state (image, position, velocity, action)
  -> HTTP POST ke Data Collector API
  -> Data disimpan sebagai frame + action
  -> Diproses ke format LeRobot
  -> Siap dipakai training di Kaggle
```

Komponen:
- Repository: https://github.com/Fth87/VLA_drone_data_collector
- Collector server: Python API listener untuk Roblox requests
- Output: dataset

## Notes

- Backend Kaggle karena untuk demo/dev saja, bukan 24/7 production.
- Frontend mendukung custom map `.glb` untuk variasi environment.

