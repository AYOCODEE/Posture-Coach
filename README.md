# Posture Coach

> Watches you through your laptop webcam and nudges you with a desktop notification when you've been slouching for 30 seconds.

**Status:** Week 2 · Day 1 (Mon): README and plan written, no app code yet. See [docs/PLAN.md](docs/PLAN.md).

## The problem
People who work at a laptop slowly sink into a slouch: head forward, shoulders rounded. It happens without them noticing, and over hours it causes neck and back pain. Reminders on a timer don't help, because they go off whether or not you're slouching. Asking someone to watch you doesn't scale. A webcam is already pointed at you, so it can do the watching.

## What it does
1. **Finds your body.** MediaPipe Pose finds 33 body landmarks (nose, ears, shoulders, hips and so on) in each webcam frame, on the CPU.
2. **Turns landmarks into posture features.** Code computes a few angles and distances, such as how far your head has dropped toward your shoulders and how far forward your neck is tilted. Each one is divided by your shoulder width, so it doesn't matter how close you sit to the camera.
3. **Classifies each frame.** A small classifier (logistic regression, with a small MLP for comparison) that I trained on my own labelled recordings gives the probability that I'm slouching.
4. **Smooths the signal.** An exponential moving average plus hysteresis (two thresholds, not one) stops the label flickering between "good" and "slouch".
5. **Shows and warns.** A live overlay draws the skeleton, the current state and the FPS. After 30 seconds of continuous slouching, it sends one desktop notification.

## Success metric
| Metric | Target | Gate? |
|---|---|---|
| Frame accuracy on a **separate** recording session (held out, recorded on a different day) | **≥ 90%** | ✅ Yes |
| End-to-end FPS on the laptop CPU (camera → pose → classify → overlay), median over 60 s | **≥ 15 FPS** | ✅ Yes |
| Flicker: smoothed state changes per minute during steady posture in the test session | **≤ 1 / min** | ✅ Yes |
| Slouch recall and precision, macro F1, raw (unsmoothed) accuracy | recorded | No |
| Alert latency: seconds from slouch start to notification | recorded (should be ~30 s plus smoothing lag) | No |

The 90% and 1/min targets are my own defaults (the brief doesn't set them). They're fixed today, before any data exists.

## Done when
> **You report accuracy on a separate recording session, the FPS, and a demo video.**

## Tech stack
- **Python 3.12**, managed with **uv**
- **MediaPipe** Pose Landmarker (Tasks API, `lite` model): landmarks on the CPU
- **OpenCV**: webcam capture and the live overlay window
- **scikit-learn**: logistic regression and MLP, `StandardScaler`, metrics
- **pandas**: CSV handling
- **plyer**: Windows desktop notifications
- **Streamlit**: the daily posture-score chart (stretch goal only)
- **pytest** and **ruff** for development

## Project rules applied here
- **A measurable eval, defined before building.** The metrics and targets above are set on Monday.
- **The eval set is a separate session.** Webcam frames a few milliseconds apart are near-duplicates, so a random train/test split of frames would leak and inflate accuracy. Test data comes from a recording made on a different day.
- **Numbers come from code.** Accuracy, FPS and flicker are computed by `eval/` scripts and saved as JSON, not read off by eye.
- **Store landmarks, not video.** Recordings save landmark coordinates to CSV. No webcam images are kept, except the demo video, which I record on purpose.
- **Reuse.** Features, smoothing and the classifier are plain modules with no UI code, so the capstone can import them.
- **One command to run:** `uv run posture-coach run`.

## Quick start
_Filled in on Thursday._

### Configuration
_Filled in on Tuesday (table: variable · default · purpose)._

## Development
_Filled in on Tuesday/Wednesday (pytest, ruff, record/train/eval commands)._

## Project structure
_Filled in on Tuesday. See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for the planned modules._

## Results
_Filled in on Friday._

| Run | Date | Model | Test accuracy | Slouch F1 | Flicker / min | FPS (median) | Done when |
|---|---|---|---|---|---|---|---|
| – | – | – | – | – | – | – | – |

Demo video: _Friday._

CV line: *Built a real-time posture-monitoring app with MediaPipe pose landmarks and a custom classifier (__% accuracy at __ FPS on CPU).*

## Stretch goal
A **daily posture score**: while the app runs it logs the share of each minute I spent in good posture. A small Streamlit page then charts the score for each day.

## Docs
- [docs/PLAN.md](docs/PLAN.md): the day-by-day plan and checklist
- [docs/EVAL.md](docs/EVAL.md): how accuracy, FPS and flicker are measured
- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md): the pipeline, modules and design decisions
