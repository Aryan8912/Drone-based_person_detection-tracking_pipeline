# Drone-based_person_detection-tracking_pipeline
<img width="2390" height="1096" alt="image" src="https://github.com/user-attachments/assets/5e7dc0bc-9768-43ec-9006-827aba9e13fb" />

## Abstract
Unmanned aerial vehicles (UAVs) are increasingly deployed for surveillance, 
crowd monitoring, and public safety applications. Detecting and tracking 
persons from a moving drone platform presents unique computer vision challenges 
including small object sizes, significant camera ego-motion, occlusions, and 
high object density. This project presents a robust, real-time multi-object 
tracking (MOT) pipeline tailored specifically for drone footage, built on 
**YOLOv8n** fine-tuned on the VisDrone dataset, combined with **ByteTrack** 
for multi-object tracking and a custom **ORB-based Ego-Motion Compensator** 
to handle drone camera movement. The system achieves **19.1 FPS** with a 
model size of only **6.3 MB**, well within edge deployment constraints, while 
delivering a **mAP@50 of 0.869** and tracking up to **51 persons simultaneously**.

---

## Requirements

- Python 3.8+
- PyTorch 2.0+
- OpenCV 4.8+
- Ultralytics YOLOv8
- lapx (ByteTrack dependency)
- CUDA-compatible GPU

```bash
pip install -r requirements.txt
```

---

## 1. Introduction

Person tracking from drone platforms poses unique challenges:

- **Small object sizes** — persons appear as 5×10 pixels in 4K footage
- **Camera ego-motion** — drone movement causes entire frame to shift
- **High density** — up to 51 persons visible simultaneously
- **Occlusions** — persons overlap and disappear behind objects
- **Scale variation** — altitude changes affect person apparent size

YOLO (You Only Look Once), introduced by Redmon et al. (2015), enables 
single-pass real-time object detection. We use **YOLOv8n** — the nano 
variant — fine-tuned specifically on VisDrone drone footage for person 
detection. For tracking, we employ **ByteTrack** with a custom two-stage 
matching strategy tuned for drone scenarios. A key contribution of this 
work is the **EgoMotionCompensator** — an ORB feature matching + RANSAC 
homography module that removes camera shake before feeding detections 
into the tracker, dramatically reducing ID switches.

---

## 2. Dataset

**VisDrone2019-MOT Validation Set** — Task 4 (Multi-Object Tracking)

| Property           | Value                        |
|--------------------|------------------------------|
| Source             | Tianjin University           |
| Sequences          | 7 video clips                |
| Total Frames       | 2,846                        |
| Resolution         | Up to 3840×2160 (4K)         |
| Frame Rate         | 25 FPS                       |
| Total Annotations  | 118,127                      |
| Person Annotations | 50,312 (42.6% of dataset)    |
| Min Person Size    | 5×10 pixels                  |

### Bounding Box Size Distribution (Persons)
| Category           | Count   | Percentage |
|--------------------|---------|------------|
| Tiny (< 32×32 px)  | 12,449  | 24.7%      |
| Small (32–64 px)   | 25,555  | 50.8%      |
| Medium (> 64 px)   | 12,308  | 24.5%      |

> 75.5% of all person annotations are tiny or small — 
> making this one of the hardest detection benchmarks.

---

## 3. Data Preprocessing

VisDrone annotations were converted from native format to YOLO normalized format:
