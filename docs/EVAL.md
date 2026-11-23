# Evaluation: Posture Coach

## Done-when test
> **You report accuracy on a separate recording session, the FPS, and a demo video.**

## Classes
| ID | Definition | Examples recorded |
|---|---|---|
| `good` | Sitting upright: ears roughly above the shoulders, chin level, shoulders back | Typing, reading the screen, small natural shifts |
| `slouch` | Any posture I'd want to be warned about | Head pushed forward, shoulders rounded, sunk low in the chair, leaning on one elbow |

The labels are defined in one place (`POSTURE_LABELS` in `src/posture_coach/features.py`). Recording, training and scoring all import them from there.

## Data
| Session | Use | Recorded | Good | Slouch | Frames |
|---|---|---|---|---|---|
| `train-01` | Training and cross-validation | Tue | ~5 min | ~5 min | _Tue_ |
| `test-01` | **Held-out test only.** Never used for training or tuning. | Thu (different day) | ~2 min | ~2 min | _Thu_ |

- Each recording is a single label (I hold one posture class for the whole clip), so the ground truth is set when I press record, not after seeing predictions.
- The first and last **3 seconds** of every clip are dropped, because that's when I'm still settling into position. This rule is fixed now and isn't tuned later.
- Frames where MediaPipe finds no person, or where shoulder visibility is below 0.5, are excluded and **counted** (reported as "frames with no usable pose").
- Model selection (LR vs MLP, regularisation, smoothing constants) uses only `train-01`, with grouped cross-validation over 30-second chunks. `test-01` is scored **once** per final model.

## CSV format (`data/sessions/<session>_<label>_<n>.csv`)
```
session,label,t_ms,frame,nose_x,nose_y,nose_z,nose_vis,l_ear_x,...,r_hip_vis
train-01,good,0,0,0.512,0.331,-0.41,0.99,...
```
Normalised image coordinates for the nose, both eyes, both ears, both shoulders and both hips.

## Scoring (all computed by code in `src/posture_coach/evaluation.py`)
| Metric | How it's computed | Target |
|---|---|---|
| **Smoothed accuracy** (headline) | Correct smoothed labels / usable frames in `test-01`, replayed in time order through the same EMA + hysteresis as the live app | **≥ 90%** |
| Raw accuracy | Same, using the per-frame classifier output with no smoothing | recorded |
| Slouch precision / recall / F1, confusion matrix | scikit-learn, positive class = `slouch` | recorded |
| **Flicker** | Smoothed state changes ÷ minutes, counted within each clip (one clip holds one true label, so every change is a flicker) | **≤ 1 / min** |
| **FPS** | `eval/benchmark_fps.py`: wall-clock time per full loop (grab frame → pose → features → classify → smooth → draw), over 60 s live after a 5 s warm-up. Report median and p5. | **median ≥ 15** |
| Alert latency | Seconds from the start of a slouch clip to the first notification event, in replay | recorded |

Unit tests cover the scorer: a perfect prediction scores 100%, all-wrong scores 0%, and a known flicker sequence gives the expected count.

## How to run
```
uv run posture-coach train --session train-01
uv run python eval/run_eval.py --session test-01
uv run python eval/benchmark_fps.py --seconds 60
```
_Commands confirmed on Friday._

## Results log
| Date | Model | Smoothed acc | Raw acc | Slouch F1 | Flicker/min | FPS median / p5 | Change made |
|---|---|---|---|---|---|---|---|
| _Fri_ | | | | | | | |

Raw JSON: `eval/results/<timestamp>_<model>.json`

Hardware: _laptop CPU model, RAM, webcam resolution (Fri)._

## Run notes
_Fri: hand review of the misclassified segments (transition frames? a slouch type missing from training? lighting?)._

## Caveats (to be confirmed with data)
- One person, one laptop, one chair. The accuracy says nothing about other people or setups.
- Self-labelled "acted" posture. Real slouching is more gradual than slouching on purpose.
- Small test set (~4 min, two clips per class), so a single bad clip moves the number a lot.
