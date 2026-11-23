# Project Plan: Posture Coach (Week 2)
**Difficulty:** Easy · **Time budget:** about 8 h across Mon–Fri
**New skills:** MediaPipe pose landmarks, real-time video loops, feature engineering from geometry, training a small classifier on self-collected data, smoothing a noisy signal.

Feeds the capstone: `features.py`, `smoothing.py` and `classifier.py` must be importable on their own, with no camera or UI code in them.

## Decisions made on Monday (change them if you disagree)
| Decision | Default chosen | Why |
|---|---|---|
| Pose API | MediaPipe **Tasks** `PoseLandmarker`, `pose_landmarker_lite.task`, VIDEO mode | Newer MediaPipe releases removed the legacy `mp.solutions.pose`. The lite model is the fastest, which helps the 15 FPS budget. Check the installed version on Tuesday. |
| Accuracy target | ≥ 90% on the held-out session | The brief gives no number. 90% is ambitious for a separate session, but realistic for a 2-class problem. |
| Flicker target | ≤ 1 state change per minute in steady posture | Makes "doesn't flicker" testable. |
| Train/test split | **By session**, never by random frames | Neighbouring frames are almost identical, so a random split would leak. |
| What gets saved | Raw landmarks (x, y, z, visibility) for the key points, not features | Features can be changed later without re-recording. |
| Commit the landmark CSVs? | Yes (small, and only coordinates) | Change this if you'd rather keep them private. |
| Live UI | OpenCV window, not Streamlit | Streamlit reruns the script and can't comfortably do 15+ FPS webcam video. Streamlit is used for the stretch chart instead. |
| Notifications | `plyer` | Pure Python and works on Windows. Swap it for `win11toast` if it misbehaves. |
| Framing | Laptop webcam, front view, hips often out of frame | Features must work with only head and shoulders visible. Hip-based features are optional. |

## Mon: Write the README first (1 h)
- [x] README: problem, success metric, done-when test
- [x] `docs/PLAN.md`, `docs/EVAL.md`, `docs/ARCHITECTURE.md`
- [ ] `git init`, first commit (you do this)

## Tue–Thu: Build the core loop end to end (5 h)

### Tue: setup, first landmarks, training recordings (~1.5 h)
- [ ] `uv init --package`, Python 3.12 (check MediaPipe supports the version), add deps: `mediapipe opencv-python scikit-learn pandas numpy joblib plyer`; dev group: `pytest ruff`
- [ ] `.gitignore`, `.env.example`, `src/posture_coach/config.py`, `errors.py`
- [ ] `scripts/download_model.py`: fetch `pose_landmarker_lite.task` into `models/` (gitignored)
- [ ] **Brief step 1:** `pose.py` + `posture-coach landmarks`: open the webcam and print the shoulder, ear and nose landmarks to the console. Note the raw FPS here.
- [ ] `recording.py` + `posture-coach record --label good|slouch --session train-01 --minutes 5`
- [ ] **Brief step 2:** record the **training session**: 5 min good, 5 min slouching. Vary the slouch (head forward, shoulders rounded, sunk low in the chair, leaning on an elbow) and shift around a little within "good", so the model learns posture rather than one exact pose.

### Wed: features, classifier, smoothing (~2 h)
- [ ] `features.py`: pure functions from landmarks to a feature vector, with `FEATURE_NAMES` as the single source of truth. Unit-tested on hand-made landmark sets (a known "upright" set and a known "head dropped" set).
- [ ] `classifier.py`: train LR and MLP, compare them with cross-validation **grouped by recording chunk** (not random frames), save the winner with `joblib` to `models/posture_clf.joblib`.
- [ ] `smoothing.py`: EMA on the slouch probability plus hysteresis (enter slouch above 0.6, leave below 0.4). Unit-tested on a synthetic noisy sequence.
- [ ] `alerts.py`: `SlouchTimer` with an injectable clock (fires once after 30 s of slouching, resets on good posture), and a `Notifier` protocol with a `plyer` implementation. Unit-tested with a fake clock and a fake notifier.
- [ ] Try it on 2–3 short live clips and check the probabilities make sense.

### Thu: live app + test session (~1.5 h)
- [ ] `overlay.py` + `app.py`: live window with the skeleton, state, slouch probability bar, slouch timer and FPS
- [ ] `posture-coach run` wired to the same feature → classifier → smoother code the eval uses
- [ ] `--record-demo out.mp4` flag that saves the overlay video (for the done-when demo)
- [ ] Record the **test session** (`test-01`, about 2 min good and 2 min slouch). This must be a different day from training, ideally different lighting and clothes.
- [ ] README: Quick start, configuration, structure

## Fri: Eval (1.5 h)
- [ ] `eval/run_eval.py`: load the saved model and the test session, report raw and smoothed accuracy, precision/recall/F1, the confusion matrix and flicker per minute, then save JSON in `eval/results/`
- [ ] `eval/benchmark_fps.py`: 60 s live run, headless timing of the full loop, report median and 5th percentile FPS
- [ ] Record the demo video (about 60–90 s: sit well → slouch → notification at 30 s → sit up)
- [ ] Fill in the README Results table, `docs/EVAL.md` and the CV line
- [ ] Review by hand: look at the misclassified frames' timestamps. Are they transition frames, or real errors?

## Stretch: daily posture score
- [ ] `history.py`: while `run` is active, append one row per minute (timestamp, % of frames good) to `data/history.csv`
- [ ] `dashboard.py` (Streamlit): bar chart of the daily score

## Rules that keep this from becoming tutorial-following
- Write the feature maths myself. Don't copy a "posture angle" snippet without understanding which landmarks it uses.
- Split by session, and say so in the results.
- Report the numbers I actually got, even if they're under target, with an honest note on why.
- No hard-coded thresholds in the app. They live in `config.py`.

## Repo layout
See [README → Project structure](../README.md#project-structure) and [ARCHITECTURE.md](ARCHITECTURE.md).

## Log
- **Mon 23 Nov:** docs written.
