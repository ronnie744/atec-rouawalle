# C.R.A.T.E.R. — Lunar Hazard Detection, Localization & Reliable Telemetry

**IEEE ATEC'26 Hackathon | Computer Vision · Spatial Localization · Communication Systems**

> From lunar imagery to actionable hazard information: detect rocks and craters, estimate their locations, and deliver compact hazard reports to Mission Control over an unreliable simulated link.

**Status:** Research and prototype notebooks with recorded experimental outputs. Dashboard integration and end-to-end verification remain in progress.

## 1. Mission and problem

Lunar rover operations require awareness of nearby terrain hazards. Visual detection alone is not sufficient: a rover also needs to estimate where hazards are and communicate useful information despite packet loss, corruption, delay, and temporary blackouts.

**C.R.A.T.E.R.** explores a modular pipeline for these tasks:

```text
Lunar images / video + rover poses
            |
            v
[1. PERCEPTION] YOLOv8n detection + tracking
            | detections.jsonl
            v
[2. LOCALIZATION] Camera-ground projection + hazard association
            | telemetry_events.json
            v
[3. TRANSPORT] Compact packets + unreliable-link simulation
            | received_hazards.json
            v
     Mission Control dashboard (integration planned)
```

The present dataset supports **two classes: crater and rock**. The transport and localization modules also contain extensible class-handling logic; that should not be interpreted as a trained ice detector.

## 2. Verified experimental highlights

### Perception — YOLOv8n validation

| Metric | Recorded value |
|---|---:|
| Precision | **0.954** |
| Recall | **0.942** |
| mAP@50 | **0.978** |
| mAP@50:95 | **0.910** |
| Validation images | **313** |
| Validation annotated objects | **957** |
| Crater AP@50 | **0.978** |
| Rock AP@50 | **0.978** |

These figures are from the **validation split**, not a separately reported held-out test-set evaluation. The notebook fine-tunes pretrained **YOLOv8n** for **60 epochs**, with **416 × 416** training inputs and batch size **16**; its saved training log identifies a **Tesla T4** GPU.

### CPU inference benchmark

| Approach | FPS | Mean latency | p95 latency | Model size |
|---|---:|---:|---:|---:|
| Classical CV baseline | 118.0 | 8.5 ms | 9.5 ms | — |
| YOLOv8n ONNX FP32 | **20.9** | **48.0 ms** | 51.1 ms | 12.68 MB |
| YOLOv8n ONNX INT8 | 11.7 | 85.5 ms | 112.4 ms | 3.39 MB |

Measured in the notebook's recorded CPU environment at 416-pixel inference size. The smaller INT8 file was **slower**, so the results do not support claiming an INT8 speedup. The classical baseline is a heuristic and its FPS should not be treated as equivalent detection accuracy. The saved video run processed **150 frames**, reporting **16.5 mean FPS** and **27 confirmed-new hazard events**.

### Localization prototype

The localization notebook's saved execution reports:

- **403** input detections across **145** frames
- **388** localized observations; **15** rejected for low confidence
- **9** tracked hazards
- **59** exported telemetry events

Localization uses camera intrinsics, assumed camera mounting geometry, ground-plane projection, rover poses, spatial association, and uncertainty estimates. **Camera parameters are assumptions**, and the notebook explicitly reports that ground truth was unavailable for real-data localization validation. Consequently, these counts are *pipeline outputs*, not a measured localization-accuracy guarantee.

### Reliable telemetry simulation

The transport notebook implements a **13-byte data packet**, CRC-based corruption detection, sequence numbering, duplicate handling, repeated priority reports, NACK-based retransmission, and heartbeats. Its simulated link can introduce loss, corruption, duplication, latency, and blackouts.

In the notebook's recorded comparisons:

| Simulated condition | Send-once baseline | NACK + heartbeat protocol |
|---|---:|---:|
| 30% packet loss — hazards known | 89% | **100%** |
| 50% packet loss — hazards known | 73% | **100%** |
| 30% loss plus a simulated 1–7 s blackout — hazards known | 50% | **100%** |

These are **simulation-specific results** and entail increased traffic and, in some cases, delay. They do not demonstrate guaranteed delivery under arbitrary conditions or on physical radio hardware.

**Integration caveat:** The transport notebook's saved execution reports loading **13 sample events**, whereas the localization notebook exported **59 events**. The full 59-event handoff must be rerun and verified before claiming a fully demonstrated end-to-end pipeline.

## 3. Dataset and attribution

**Lunar Rocks and Craters Dataset for Instance Segmentation (YOLO and COCO Format)**, Siddhant Shete, Zenodo v1 (2025). DOI: [10.5281/zenodo.14673237](https://doi.org/10.5281/zenodo.14673237).

- **Source:** [Zenodo record](https://zenodo.org/records/14673237)
- **Selected archive:** `Dataset_yolo.zip`
- **Classes:** rock and crater
- **Published splits:** 7,734 train / 313 validation / 307 test images
- **Published image size:** 640 × 640 pixels
- **License:** Creative Commons Attribution 4.0 (CC BY 4.0)

The dataset was collected at the German Research Center for Artificial Intelligence (DFKI), Robotics Innovation Center, according to the dataset record. The dataset provides segmentation annotations; this prototype uses a **YOLO object-detection workflow**. Credit the dataset authors when redistributing or reusing licensed material.

## 4. Notebook responsibilities

| Notebook | Responsibility | Outputs / artifacts |
|---|---|---|
| `crater_perception (2).ipynb` | Dataset checks, classical CV baseline, YOLOv8n fine-tuning, validation, ONNX export, CPU benchmarking, detection/tracking | `perception.py`, `detections.jsonl`, ONNX weights, benchmark CSV, annotated video |
| `hazard_localization.ipynb` | Detection normalization, pixel-to-ground projection, rover-frame/world-frame mapping, uncertainty, multi-frame hazard association | `telemetry_events.json` |
| `telemetry_transport.ipynb` | Compact packet encoding, lossy-link simulation, retransmission/heartbeat comparisons | `received_hazards.json` |

**Recommended execution order:** perception → localization → transport. Verify the file paths and data contracts between stages; the notebooks currently include fallback/synthetic or sample-data paths that must not be confused with a verified integrated run.

## 5. Reproducing the experiments

1. Obtain `Dataset_yolo.zip` from the cited Zenodo record and respect its CC BY 4.0 license.
2. Open the perception notebook in a compatible Python/Jupyter environment (its saved run uses Google Colab paths under `/content/`).
3. Install the dependencies used by the notebooks, including `ultralytics`, `onnxruntime`, `onnx`, `psutil`, `lapx`, `pyyaml`, `numpy`, `pandas`, `matplotlib`, and `opencv-python` as appropriate.
4. Update dataset and output paths for your environment, then execute the perception notebook in order. The notebook includes a training step; **do not rerun it unintentionally** if you only want to inspect existing outputs.
5. Provide detections and matching rover poses to the localization notebook. Review camera configuration before interpreting positions in metres.
6. Point the transport notebook to the actual `telemetry_events.json` and **verify that it loads the intended event count**, rather than falling back to built-in sample events.
7. Inspect `received_hazards.json` and the notebook-generated plots and benchmark results.

For reproducible jury review, add a pinned `requirements.txt`, tested run commands, small example inputs, and a manifest of artifacts once these are finalized. Avoid committing full datasets, credentials, large generated videos, and oversized weights directly to the repository.

## 6. Engineering decisions and limitations

- **Small edge model:** YOLOv8n and ONNX CPU inference were evaluated for constrained compute. The notebook also evaluates a classical CV fallback.
- **Localization uncertainty:** Projection uncertainty is modeled, but physical camera calibration and independent position ground truth are not established in the saved run.
- **Communications resilience:** The custom protocol is tested under a configurable *simulator*, not a physical rover radio.
- **Dataset scope:** The labeled dataset covers **rocks and craters**, not every possible lunar hazard.
- **Generalization:** High validation metrics do not establish robustness across unseen lunar missions, lighting conditions, sensors, or a separately evaluated test split.
- **Dashboard:** A Mission Control visualization interface is planned; live integration should only be marked complete after it is demonstrated.

## 7. Next milestones

- [ ] Run and report a separate held-out **test-split evaluation**.
- [ ] Calibrate camera geometry and quantify localization error against ground truth.
- [ ] Demonstrate the complete **59-event localization → transport** handoff.
- [ ] Connect `received_hazards.json` or a live event stream to the Mission Control dashboard.
- [ ] Add detection images, plots, demo screenshots, and a short walkthrough.
- [ ] Pin dependencies, publish tested reproduction instructions, and document team contributions.

## 8. Team and credits

**IEEE ATEC'26 Hackathon — C.R.A.T.E.R. team**  
Team members and module ownership: *add verified names / GitHub handles before submission*.

**Dataset:** Shete, S. (2025). *Lunar Rocks and Craters Dataset for Instance Segmentation (YOLO and COCO Format).* Zenodo. https://doi.org/10.5281/zenodo.14673237

**Libraries and tools:** Ultralytics YOLO, ONNX Runtime, OpenCV, NumPy, pandas, Matplotlib, Python, and Jupyter.

---

*Detect locally. Localize intelligently. Communicate reliably.*
