# IRobots

A robotic arm you talk to instead of program.

## Problem statement

Industrial arms are taught one job and repeat it forever. Changing the job
means someone rewrites the motion sequence, teaches every waypoint again,
and takes the cell offline for a day. That is the wrong shape for how
small factories, labs, and workshops actually operate, where the task
changes every week and nobody on staff is a robotics engineer.

IRobots is an attempt at a controller layer that sits on top of an arm and
takes a plain-language task instead of a motion program: describe the job,
and the same hardware does a different job with no new code. This
repository is our hackathon submission for that idea: mechanical design,
camera firmware, a perception pipeline, and an LLM tool-calling agent that
issues movement commands one atomic step at a time.

## Where the project actually stands

We would rather tell you what runs than describe what we intended to
build. This table reflects the code in this repository as of the
submission, not the design.

| Piece | Status | What that means concretely |
|---|---|---|
| ESP32 camera node | Working | Streams MJPEG and stills over Wi-Fi. Flash it and it runs. |
| Arm mechanical design | Working | Five-joint SolidWorks assembly under `robotic arm schematics/`. |
| Arm safety firmware | Working | E-stop with a debounced two-press resume, joint-angle clamping, an RGB status light, a buzzer, and an 8x8 status matrix all run today on the microcontroller. |
| Arm networked control | Not started | The arm-side firmware currently runs a fixed, hardcoded sweep of preset angles (`src/micro_controller/ardunio/V1_arm_controller_full_status.ino`). It does not yet receive commands over the network, does not solve inverse kinematics, and has no watchdog for dropped commands. This is the piece the rest of the pipeline assumes exists and does not yet. |
| Vision and coordinate mapping | In progress | SAM 2.1 segmentation and a pixel-to-centimetre scale factor are implemented in `sam2_processor.py`. The scale factor is a single constant (`PIXELS_PER_CM`) rather than a calibrated four-point homography, so it is closer to a first pass than the calibrated system described in `docs/hardware.md`. |
| Agent layer | Partially working, single-model | `agent_harness.py` streams a single vision-capable model's reasoning and tool calls, enforces one tool call per turn, and validates arguments against hardware bounds before reporting success. It has been run end to end against a static test image; it has not been run against the live camera or the live arm. |
| Planner / worker split | Designed, not built | The two-model architecture described below is the design target. Only one model exists in the code today; `orchestrator.py`'s outer loop re-invokes it turn by turn in place of a separate planning model. |
| Hardware dispatch | Not connected | `validate_and_execute_tool` in `agent_harness.py` checks a proposed move against the workspace bounds and returns a success or error message, but nothing in the repository sends that command over the network to the arm controller. The loop today is camera to detector to model to a validated-but-undelivered instruction. |
| Operator interface | Working, terminal only | A `rich`-based TUI in `orchestrator.py` runs the pipeline interactively. |

If a demo needs to show "one continuous prompt-to-motion loop," that
specific claim is not yet true end to end. What is true and demoable
today: the camera node, the mechanical arm with its safety firmware, and
the perception-to-tool-call reasoning loop running against a static image
or a live camera frame, with its output validated but not yet delivered
to a servo.

## How the system is designed to fit together

```
                 operator types a task
                          │
                          ▼
             ┌────────────────────────┐
             │     planner model      │   turns the task into ordered steps
             └────────────────────────┘        (designed, not yet built)
                          │
   camera frames          ▼
        │    ┌────────────────────────┐
        └───▶│    perception layer    │   detection, then pixels to centimetres
             └────────────────────────┘        (partly built)
                          │
                          ▼
             ┌────────────────────────┐
             │      worker model      │   picks the target, emits one tool call
             └────────────────────────┘        (built, run against static images)
                          │
                          ▼   JSON over Wi-Fi
             ┌────────────────────────┐
             │     arm controller     │   inverse kinematics, joint limits, motion
             └────────────────────────┘        (not built; firmware runs a fixed
                          │                      demo sequence today)
                          ▼
                   the arm moves, and
                 the outcome is reported
                     back to the planner
```

The loop matters more than any single box: the worker acts, the camera
sees the outcome, and the planner decides whether the step succeeded.
Right now the last two boxes and the feedback edge are the gap between
this diagram and a running system. A longer walkthrough of the intended
design lives in [docs/architecture.md](docs/architecture.md); read the
table above alongside it, since the doc describes the target, not
uniformly what is built.

## How this compares to vision-language-action models, honestly

The obvious approach to "camera in, motor commands out" is a
vision-language-action (VLA) model: fine-tune a vision-language backbone
end to end on (image, instruction, action trajectory) data so it directly
emits joint commands. That is an active, fast-moving research area right
now, with model lines like RT-2, OpenVLA, and pi-0 doing exactly that in
a single forward pass, and dual-system designs like Helix and GR00T N1
splitting a slower "understand the scene" component from a faster "emit
actions" component.

We are not doing that, and we want to be precise about why, because
"we don't need a VLA" is a real tradeoff, not a strict improvement.

We use a general-purpose vision-language model for reasoning and tool
selection, and keep every spatial coordinate coming from a deterministic
perception step rather than from the model. The model picks an object by
identity from a list perception already measured; it never estimates a
position. This is close to the agentic tool-calling line of work in
robotics -- SayCan and Code as Policies established the pattern of an LLM
selecting from or composing calls to a fixed skill library rather than
emitting raw actions, and recent 2026 work under names like
"VLA-as-tools" or tool-aligned VLA agents is actively formalizing exactly
this planner-plus-tool-call pattern for long-horizon manipulation. We are
applying that established pattern to a specific, low-cost physical build,
not inventing a new one. That is a meaningfully smaller and more honest
claim than "our architecture beats current VLA architecture," and it is
the claim we can actually defend if a judge who works in this space asks
about it.

What this approach actually buys, and what it costs:

* **No training run, no dataset to collect.** True, and it is the main
  reason a small team can attempt this at all inside a hackathon. A
  fine-tuned VLA needs a demonstration dataset for the target embodiment;
  we get task generality from prompting and a small tool schema instead.
* **Generalizes to new object classes without retraining, so long as
  perception can detect them and the prompt can describe the task.** This
  is real, and it is the correct thing to say generalizes -- not the
  whole system. Detection is still the ceiling: an open-vocabulary
  detector is on the roadmap and is currently not built, so today the
  system generalizes across tasks more than it generalizes across object
  classes.
* **The tradeoff we are not currently paying down: latency and learned
  visuomotor precision.** A general-purpose LLM call in the loop, even
  restricted to one tool call per turn, is meaningfully slower per action
  than a small trained policy's forward pass, and it has no learned sense
  of contact dynamics or force -- it is reasoning over coordinates it was
  handed, not perceiving through touch. A fine-tuned VLA trades away
  generality to buy exactly that precision and speed. We are betting that
  for a demo-scale set of pick-and-place tasks, the coordinate-only rule
  and hardware-side clamping cover the gap; that bet has not been
  validated against a live arm yet.

## Repository layout

```
docs/                          project documentation
pics/                          photos and captures for the writeup
presentation/                  slides and demo material
robotic arm schematics/        SolidWorks assembly for the arm
src/
  micro_controller/
    esp32/                     arm side firmware (placeholder; not started)
    ardunio/                   arm controller sketch, safety firmware, and its tests
  vision-AI/
    vison/                     ESP32 camera web server firmware
    config/                    calibration and runtime configuration
    simulation/                offline testing without hardware
```

Directories holding a lone `temp.txt` are placeholders for work in
flight. They exist so the structure is settled before the code lands.

## Running the camera node

The ESP32 camera node is the piece that runs today. Full detail is in
[docs/camera_server.md](docs/camera_server.md); the short version:

1. Install the Arduino IDE and add the `esp32` board package from Espressif.
2. Open `src/vision-AI/vison/CameraWebServer.ino`.
3. Copy `camera_index.h` from the Arduino CameraWebServer example into the same folder. It is deliberately kept out of version control because it is a large generated blob, and the sketch will not compile without it.
4. Put your network name and password at the top of the sketch.
5. Select board **AI Thinker ESP32 CAM**, set PSRAM to enabled, and set the partition scheme to **Custom** so the `partitions.csv` in the sketch folder is used.
6. Tie GPIO0 to ground, press reset, and upload. Remove the jumper and press reset again when it finishes.
7. Open the serial monitor at 115200 baud and read off the address the board prints.

Once it is up, put that address in a shell variable and pull a frame:

```bash
export BOARD_IP=192.168.1.50
```

```bash
curl http://$BOARD_IP/capture --output frame.jpg
```

The browser interface is at `http://$BOARD_IP/`, and the raw MJPEG stream
the vision pipeline consumes is at `http://$BOARD_IP:81/stream`.

## Running the Python pipeline

```bash
pip install -r requirements.txt
python orchestrator.py --offline   # runs against test_artifacts/, no camera or arm required
```

`--offline` is currently the only mode that has actually been exercised.
Dropping it requires a reachable ESP32 camera node and does not yet
require or use a connected arm controller, since nothing in the pipeline
sends commands to one (see the status table above).

## Hardware

| Part | What we used | Count |
|---|---|---|
| Arm | Five joint design, SG90 class servos | 1 |
| Camera board | ESP32 CAM, AI Thinker layout with OV2640 | 1 |
| Arm controller | ESP32 development board | 1 |
| Servo supply | 5V at 2A or better, separate from the boards | 1 |
| USB serial adapter | For flashing the camera board | 1 |
| Workspace | Flat surface with even lighting and four fixed calibration markers | 1 |

One thing worth repeating because it has bitten us: do not run the
servos off the ESP32 regulator. Give them their own supply and tie the
grounds together. Wiring notes and the rest of the setup are in
[docs/hardware.md](docs/hardware.md).

## What we would build next, in order

Not a wishlist -- the order we would actually spend the next block of
time in, because each item is a prerequisite for the one after it:

1. **Wire the arm controller to the network.** Give
   `V1_arm_controller_full_status.ino` a JSON-over-Wi-Fi listener so it
   can receive `(x, y, z)` instead of running its hardcoded sweep. Nothing
   else on this list matters until a command can reach the arm.
2. **Close the loop end to end, once, on the live arm.** One task,
   one object, from typed instruction to a completed grasp, with a human
   ready on the e-stop. This is the point where the coordinate-only rule
   and the hardware bounds checking either hold up against real
   calibration error or they do not.
3. **Calibrated homography, replacing the single `PIXELS_PER_CM`
   constant.** The four-marker calibration procedure is written up in
   `docs/hardware.md`; the code to build and apply the transform from it
   does not exist yet.
4. **The planner model**, as a second call distinct from the worker,
   with the tool-call history and outcomes fed back to it as context.
5. **Depth from a second camera angle**, removing the "guess object
   height from its class" step, which is the single largest source of
   grasp failure in the current design.

The full longer-term list, including a ROS 2 port, multi-arm
coordination, and open-vocabulary detection, is in
[docs/roadmap.md](docs/roadmap.md).

## Documentation

* [docs/README.md](docs/README.md) is the index.
* [docs/architecture.md](docs/architecture.md) covers the design and the reasoning behind it. Read it as design intent, not a description of what is built; the status table above is the source of truth for what runs today.
* [docs/hardware.md](docs/hardware.md) covers the build, wiring, and calibration.
* [docs/camera_server.md](docs/camera_server.md) covers the camera firmware and its HTTP interface.
* [docs/roadmap.md](docs/roadmap.md) covers what we are building next and in what order.
* [Plan, TODO, Idea, Execution overview](<docs/Plan - TODO - Idea -  Execution overview.md>) is the original working notes, kept as written.
* [FIXES.md](FIXES.md) is a punch list of concrete repository issues (a broken import, missing dependency file, duplicated firmware, and others) with the exact patch for each, written for cleanup before submission.

## Team

Built by **IRobots**. The work is split across mechanical design,
firmware, the vision and agent pipeline, and the demo itself, though at
hackathon hours everyone ends up touching everything.

## License

Released under the MIT License. See [LICENSE](LICENSE).

The camera firmware under `src/vision-AI/vison/` derives from Espressif's
CameraWebServer example and stays under Apache 2.0. Attribution is in
[NOTICE](NOTICE).
