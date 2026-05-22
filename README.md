#  Dronw Based Person Detection Tracking Pipeline

### Base Model: YOLOv8n (Nano)
- Chosen for ultra-lightweight size (6.3 MB vs 300 MB limit)
- Fast inference (~20 FPS on A100)
- Strong baseline mAP for nano-scale model
<img width="2390" height="1096" alt="image" src="https://github.com/user-attachments/assets/5e7dc0bc-9768-43ec-9006-827aba9e13fb" />

> Fine-tuned YOLOv8n + ByteTrack + Ego-Motion Compensation  
> Built for VisDrone Dataset | Optimized for Edge Deployment

---

## 📽️ Output  Preview

<img width="1526" height="890" alt="image" src="https://github.com/user-attachments/assets/0f9059aa-08ed-45c7-94bc-f22c6e230393" />


---

## Personal tracked per frame

<img width="1589" height="789" alt="image" src="https://github.com/user-attachments/assets/2d9264c5-f169-4e71-8d1c-b7f5a9407497" />

---

## 📊 Sample Tracked Frames

<img width="2362" height="1377" alt="image" src="https://github.com/user-attachments/assets/e7693b0f-ea00-4e0a-bc2a-8fb3072c8257" />


---

## 📈 Training Results

<img width="1790" height="928" alt="image" src="https://github.com/user-attachments/assets/1ab0f1d5-93ea-439a-afa9-e17ff71715a9" />


---

## 📊 Performance Metrics

| Metric              | Value            |
|---------------------|------------------|
| **mAP@50**          | 0.869            |
| **mAP@50-95**       | 0.486            |
| **Precision**       | 0.921            |
| **Recall**          | 0.876            |
| **Mean FPS**        | 19.1             |
| **Model Size**      | 6.3 MB           |
| **Max Persons**     | 51 / frame       |
| **Ego-motion**      | 300+ inliers     |
| **Hardware**        | NVIDIA A100 40GB |

---

### Why YOLOv8n over larger models?
┌─────────────┬──────────┬──────────┬────────────┐
│ Model       │ Size     │ FPS      │ mAP@50     │
├─────────────┼──────────┼──────────┼────────────┤
│ YOLOv8n     │ 6.3 MB   │ ~20 FPS  │ 0.869      │
│ YOLOv8s     │ 22 MB    │ ~15 FPS  │ ~0.88      │
│ YOLOv8m     │ 52 MB    │ ~10 FPS  │ ~0.90      │
│ YOLOv8x     │ 131 MB   │ ~5 FPS   │ ~0.92      │
└─────────────┴──────────┴──────────┴────────────┘
→ YOLOv8n gives best speed/accuracy tradeoff for drone edge use

### Small Object Detection Strategy
1. Fine-tuned on VisDrone dataset (drone-specific)
   - 50,312 person annotations
   - 75.5% tiny/small objects (< 64×64 px)

2. Drone-specific augmentations during training:
   - degrees=15   → simulates drone tilt
   - scale=0.9    → simulates altitude variation  
   - hsv_v=0.4    → simulates lighting changes
   - mosaic=1.0   → simulates crowded scenes

3. Higher box loss weight (box=7.5) for small objects
4. Low confidence threshold (conf=0.25) to catch tiny persons
5. imgsz=640 balances speed vs small object visibility


## 2. TRACKING — ID SWITCHING PREVENTION
----------------------------------------------------------------

### ByteTrack Configuration (drone-tuned)
- track_high_thresh : 0.35  (lower than default 0.5)
- track_low_thresh  : 0.10  (catch occluded persons)
- track_buffer      : 45    (1.8 sec memory @ 25fps)
- match_thresh      : 0.75  (strict IoU matching)
- fuse_score        : True  (detection score fusion)

### How We Handle ID Switching

Problem 1 — Drone Ego-Motion:
- Drone moves → entire frame shifts
- ByteTrack thinks ALL persons moved
- IoU between predicted/detected boxes drops → ID switch

Solution - EgoMotionCompensator:
- ORB feature matching between consecutive frames
- RANSAC homography estimation (removes moving persons)
- Compensate detection boxes using inverse homography
- Result: stable box positions despite camera movement
- Achieved 300+ inlier matches per frame (very high confidence)

Problem 2 — Occlusion:
- Persons overlap → detector misses one
- Track lost → new ID assigned on reappearance

Solution → ByteTrack two-stage matching:
- Stage 1: high-conf detections matched with Kalman prediction
- Stage 2: low-conf detections matched with lost tracks
- track_buffer=45: keeps lost track alive for 1.8 seconds
- Result: person reappears → same ID restored

Problem 3 — Scale Variation (altitude changes):
- Drone ascends/descends → person size changes rapidly
- Kalman filter scale prediction may fail

Solution - scale augmentation during training (scale=0.9)
- Model robust to size variations
- Kalman filter handles gradual scale changes


## 3. OPTIMIZATION & FPS
----------------------------------------------------------------

### Hardware Used for Testing
- GPU  : NVIDIA A100 40GB (Modal.com cloud)
- CPU  : 6.63 cores
- RAM  : 19.67 GB
- VRAM : 14.94 GB used / 40 GB available

### FPS Results
- Mean FPS    : 19.1
- Max FPS     : 21.9
- Min FPS     : 10.6 (warmup frame 1)
- Stable FPS  : ~19-21 after warmup

### Speed Breakdown (per frame ~52ms)
┌─────────────────────┬────────────┐
│ Component           │ Time (ms)  │
├─────────────────────┼────────────┤
│ Ego-motion (ORB)    │ ~3 ms      │
│ YOLOv8n inference   │ ~40 ms     │
│ ByteTrack           │ ~2 ms      │
│ Visualization       │ ~7 ms      │
├─────────────────────┼────────────┤
│ TOTAL               │ ~52 ms     │
└─────────────────────┴────────────┘

### Model Size Budget
┌─────────────────────┬────────────┐
│ Component           │ Size       │
├─────────────────────┼────────────┤
│ YOLOv8n weights     │ 6.3 MB     │
│ ByteTrack           │ 0 MB       │
│ Ego-motion (ORB)    │ 0 MB       │
├─────────────────────┼────────────┤
│ TOTAL               │ 6.3 MB     │
│ LIMIT               │ 300 MB     │
│ BUDGET USED         │ 2.1%       │
└─────────────────────┴────────────┘


## 4. EDGE DEPLOYMENT — NVIDIA JETSON
----------------------------------------------------------------

### Recommended Hardware
- Jetson Orin NX 16GB  → best balance
- Jetson AGX Orin      → maximum performance

### Deployment Steps

Step 1 — Export to TensorRT:
  model.export(format='engine',
               device=0,
               half=True,      # FP16
               imgsz=640)
  → Converts best.pt → best.engine
  → TensorRT optimizes for Jetson GPU
  → Expected speedup: 2-3x

Step 2 — INT8 Quantization (optional):
  model.export(format='engine',
               int8=True,
               data='data.yaml')
  → Further 2x speedup
  → Slight accuracy drop (~2% mAP)
  → Good for Jetson Nano class devices

Step 3 — Expected Jetson FPS:
┌──────────────────┬──────────────┬────────────┐
│ Device           │ FP16 FPS     │ INT8 FPS   │
├──────────────────┼──────────────┼────────────┤
│ Jetson Nano      │ ~8 FPS       │ ~15 FPS    │
│ Jetson Orin NX   │ ~20 FPS      │ ~35 FPS    │
│ Jetson AGX Orin  │ ~35 FPS      │ ~60 FPS    │
└──────────────────┴──────────────┴────────────┘

Step 4 — Additional Optimizations:
  - Use DeepStream SDK for multi-stream pipeline
  - Reduce imgsz to 416 for Nano class devices
  - Use ONNX Runtime with CUDA EP as fallback
  - Enable CUDA unified memory for large frames


## 5. FINAL METRICS
----------------------------------------------------------------

Detection:
  mAP@50      : 0.869
  mAP@50-95   : 0.486
  Precision   : 0.921
  Recall      : 0.876

Tracking:
  Mean persons/frame : 40.6
  Max  persons/frame : 51
  Ego-motion success : >99%
  Inliers/frame      : 300+

Performance:
  Mean FPS    : 19.1
  Model size  : 6.3 MB
  Hardware    : NVIDIA A100 40GB

================================================================
