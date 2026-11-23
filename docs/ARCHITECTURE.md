# Architecture: Posture Coach

## Pipeline
```
 webcam frame (OpenCV)
        │
        ▼
 PoseEstimator ── MediaPipe PoseLandmarker (lite, VIDEO mode, CPU)
        │  Landmarks (nose, eyes, ears, shoulders, hips: x, y, z, visibility)
        ▼
 features.extract() ── pure maths → feature vector (FEATURE_NAMES)
        │
        ▼
 PostureClassifier.predict_proba() ── scikit-learn Pipeline (scaler + LR/MLP)
        │  p(slouch) for this frame
        ▼
 Smoother ── EMA + hysteresis → stable state: good | slouch
        │
        ├──► SlouchTimer ── 30 s continuous slouch → Notifier (plyer), once
        ├──► overlay.draw() ── skeleton, state, probability bar, timer, FPS
        └──► HistoryLogger ── per-minute % good → data/history.csv (stretch)
```
Recording uses the same first two boxes and writes landmarks to CSV. The eval replays CSVs through `features → classifier → Smoother`, so the eval and the live app share one code path.

## Planned features (front-facing laptop webcam)
All distances are divided by **shoulder width**, so moving closer to or further from the camera doesn't change them.

| Feature | What it captures |
|---|---|
| `head_drop` | Vertical gap between the ear midpoint and the shoulder midpoint. It shrinks as the head sinks forward and down. |
| `nose_drop` | Vertical gap between the nose and the shoulder line |
| `neck_angle` | Angle of the ear→shoulder line from vertical (per side, averaged over the visible sides) |
| `shoulder_width_ratio` | Shoulder width compared with face width (ear to ear). Rounded shoulders narrow it. |
| `head_roll` / `shoulder_tilt` | Tilt of the eye line and the shoulder line (leaning on one elbow) |
| `nose_z_rel` | MediaPipe's depth estimate of the nose relative to the shoulders (head pushed forward) |
| `torso_angle` | Shoulder→hip angle, **only if hips are visible**. Otherwise it's left out (check on Wed whether it's worth keeping). |

These are hypotheses. On Wednesday, check which ones actually separate the two classes, and drop the ones that don't.

## Modules
| File | Responsibility |
|---|---|
| `src/posture_coach/config.py` | Frozen `Settings` dataclass (camera index, model paths, thresholds, alert seconds, EMA alpha). The only place env vars are read. |
| `errors.py` | Exception hierarchy rooted at `PostureCoachError` |
| `pose.py` | `PoseEstimator` protocol plus the MediaPipe implementation. Returns `Landmarks` or `None`. |
| `features.py` | `POSTURE_LABELS`, `FEATURE_NAMES`, `extract(landmarks) -> np.ndarray`. Pure functions, no I/O. |
| `recording.py` | Capture loop → CSV, and the CSV loader (returns DataFrames) |
| `classifier.py` | `train()`, `PostureClassifier.load()/predict_proba()`, grouped cross-validation |
| `smoothing.py` | `Smoother` (EMA + hysteresis), stateful, one `update(p) -> state` per frame |
| `alerts.py` | `SlouchTimer` (injectable clock), `Notifier` protocol, `PlyerNotifier` |
| `overlay.py` | OpenCV drawing only |
| `app.py` | Live loop, FPS meter, optional demo-video writer |
| `evaluation.py` | Accuracy, F1, flicker and alert-latency scoring (unit-tested) |
| `cli.py` | `posture-coach landmarks / record / train / run` |
| `history.py` (stretch) | Per-minute score log |
| `dashboard.py` (stretch) | Streamlit daily chart |
| `eval/run_eval.py`, `eval/benchmark_fps.py` | Thin CLIs that save JSON results |

## Errors
| Exception | When it's raised | What the user sees |
|---|---|---|
| `CameraError` | Webcam can't be opened, or returns no frames | "Couldn't open camera 0. Is another app using it? Try `POSTURE_CAMERA_INDEX=1`." |
| `ModelNotFoundError` | `.task` file or trained classifier missing | "Pose model missing: run `uv run python scripts/download_model.py`", or "Classifier missing: run `posture-coach train` first." |
| `DataError` | Empty session, only one label present, wrong CSV columns | A message naming the file and the problem |
| *(not an error)* No person in frame | MediaPipe returns no landmarks | The overlay shows "No person detected". The smoother and timer hold their state rather than counting this as good or slouch. |

## Design principles
- **One code path.** Eval replay and live use call the same `features → classifier → Smoother`.
- **Save raw landmarks.** Features are recomputed from them, so feature ideas can change without re-recording.
- **Inject time and side effects.** The timer takes a clock, and notifications go through a protocol, so a 30-second alert is tested in milliseconds.
- **Keep the classifier small and explainable.** Logistic regression first. The MLP must beat it on grouped cross-validation to be chosen.
- **Hold state when no person is detected.** Walking away from the desk shouldn't trigger or reset a slouch alert.

## Performance budget (15 FPS = 66 ms per frame)
Expected: the lite pose model takes about 20–40 ms on a laptop CPU, and features, classification and drawing take under 2 ms. So the camera's own frame rate (often 30 FPS, sometimes lower in dim light) is the likely limit. The camera is opened at 640×480. Measured numbers go in EVAL.md on Friday.
