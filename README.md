# SURGEBOT: Voice-Activated Surgical Tool Delivery Arm

A 6-DOF robotic arm that listens for a spoken request ("SURGEBOT, provide scalpel"), finds the requested instrument on a tray using a custom-trained object detector, and moves to pick it up and present it using inverse kinematics.

[![Demo video](images/demo-thumbnail.jpg)](https://www.youtube.com/watch?v=Ohq5c7Ah-D8)


**Demo video:** https://www.youtube.com/watch?v=Ohq5c7Ah-D8

> **Disclaimer:** SURGEBOT is an undergraduate prototype built as a proof of concept. It has not been tested in a clinical or sterile environment, is not a medical device, and must not be used in real procedures.

*Undergraduate minor project, Manipal Institute of Technology (MAHE), November 2025.*
*Team: Unarv Erinjery Bijumon, Nathan Ronald Gonsalves, Juliyan Deepak. Guide: Dr. Amritanshu Gupta.*

---

## Motivation

During surgery, scrub nurses spend much of their time handing instruments to the surgeon. Automating that hand-off could free staff for patient monitoring and reduce the number of people touching the tool trolley and sterile field, which lowers contamination risk. SURGEBOT is not meant to replace the surgeon or the support staff, only to assist them.

## How it works

```mermaid
flowchart LR
    A[Microphone] --> B[Speech recognition]
    B --> C[Command parsing<br/>wake phrase + tool name]
    C --> D[Object detection<br/>on webcam frames]
    D --> E[2D box to workspace<br/>coordinates x, y, z]
    E --> F[Inverse kinematics]
    F --> G[Trajectory smoothing]
    G -->|joint angles over serial| H[Arduino]
    H -->|PWM| I[6x MG995 servos]
```

1. **Speech:** a microphone on the Raspberry Pi 5 listens for a key phrase followed by an instrument name. The speech is transcribed and parsed to identify the requested tool.
2. **Vision:** a pole-mounted Logitech webcam streams the tool tray. A YOLO-based detector, trained on a custom surgical-tool dataset in COCO annotation format, returns class labels and bounding boxes.
3. **Localization:** the bounding box of the requested tool is converted from pixel coordinates to workspace coordinates using the camera calibration and a perspective transform.
4. **Motion:** the inverse kinematics solver computes the six joint angles, and an interpolating trajectory generator produces smooth intermediate angles. Forward kinematics is used to check the commanded pose (reachability and exclusion zones).
5. **Actuation:** the Pi sends the joint angles over serial to an Arduino, which only generates the PWM signals for the six servos.

## Hardware

| Component | Details |
|---|---|
| Arm | Generic commercial 6-DOF arm (assembled kit) |
| Servos | 6x Tower Pro MG995, 180 degrees, 4.8 to 7.2 V, about 15 kg-cm stall torque |
| Battery | 7.4 V (2S) 2000 mAh 30C Li-Po |
| Regulator | UBEC set to 7.4 V (8 A continuous, 16 A burst) |
| Power board | Custom perfboard PDB: diode drop stage, 4700 uF bulk capacitors, per-servo decoupling |
| High-level controller | Raspberry Pi 5 |
| Servo controller | Arduino (PWM generation only) |
| Camera | Logitech USB webcam on a 3D-printed mount and pole |
| Microphone | External USB microphone |

<img width="1280" height="960" alt="WhatsApp Image 2025-11-05 at 3 44 31 PM" src="https://github.com/user-attachments/assets/02f4a821-ebce-48f3-975d-f6400a0fc0ef" />


### Power system

```
Li-Po 7.4V -> UBEC (7.4V) -> 2 parallel branches of 3x 1N5408 in series (~2.1V drop)
           -> 4700uF capacitors -> 6 servo lines (each with its own decoupling cap)
```

The MG995's recommended maximum is 7.2 V, so the UBEC output sits just above the servo range. Three series 1N5408 diodes per branch drop roughly 0.7 V each, giving an estimated servo supply of about 5.3 V. The bulk capacitors absorb current spikes when several servos start at once.

<img width="960" height="1280" alt="WhatsApp Image 2025-11-05 at 3 44 31 PM (1)" src="https://github.com/user-attachments/assets/9783a2d3-bba0-4349-9061-dc9b2d06d4f2" />
<img width="1280" height="960" alt="WhatsApp Image 2025-11-05 at 3 44 31 PM (2)" src="https://github.com/user-attachments/assets/e9d9c7b8-c1e3-4822-a2fc-fe93f3213e45" />
<img width="1280" height="960" alt="WhatsApp Image 2025-11-05 at 3 44 31 PM (3)" src="https://github.com/user-attachments/assets/7d0d663d-101c-4109-a20d-71cc4b5e82a2" />





## Computer vision results

Training ran for 300 epochs. Average precision at IoU 0.5 (mAP50) per class, read from the validation plot:

| Class | mAP50 (%) | Class | mAP50 (%) |
|---|---|---|---|
| Weitlaner | 89 | Mayo_metz | 68 |
| Hemostat | 78 | Yankauer | 67 |
| Richardson | 78 | Bulldog | 66 |
| Needle | 76 | Scalpel | 64 |
| Frazier | 72 | Army_navy | 60 |
| Towel_clip | 71 | Potts | 52 |
| Forceps | 51 | Castroviejo | 31 |
| Iris | 28 | **All classes** | **63** |




Large instruments with distinctive shapes are detected reliably. Small, shiny, or similar-looking tools (Iris, Castroviejo) perform poorly, likely because of glare, similarity to other classes, and few pixels per object. Retrieval of those tools would need extra visual confirmation.

**Dataset:** ["Surgical tools_last.version"](https://universe.roboflow.com/uba-vbsxi/surgical-tools_last.version) by *uba*, Roboflow Universe, licensed CC BY 4.0. The source page lists 2,998 images and 14 classes. <!-- 


## System performance

**Informal end-to-end testing:** in demonstration runs of the full voice-to-pick pipeline, roughly 6 out of 10 attempts succeeded (about 4 failed). This figure is the team's recollection of informal testing and was not recorded in a systematic trial log, so treat it as approximate. Failure causes were not logged individually. The likely contributors are the servo backlash and jitter and the weaker detection of some tool classes described below.

Systematic measurements (success rate per tool, placement error in mm, and command-to-motion latency) have not been collected yet and are planned as future work.

The test run is shown in `trialrun1.mp4` and the demo video above.

## Known limitations

- **Servo accuracy:** the MG995s have noticeable backlash and jitter. They are open-loop hobby servos with no position feedback, so forward kinematics here is computed from commanded angles and cannot detect actual joint error. Closed-loop actuators (stepper or brushless with encoders) would be the proper fix.
- **Diode-based voltage drop:** a diode's forward drop changes with current, so the servo voltage varies with load. A buck converter set to about 6 V would regulate better and waste less power.
- **Localization:** a single fixed camera gives 2D positions; converting to x, y, z relies on the tools lying on a known plane. <!-- Confirm this assumption -->
- **Weak classes:** Iris and Castroviejo are detected poorly (see results above).
- **No force sensing:** the gripper cannot sense grip force or slipping.
- **Not sterile or clinical:** no sterile draping, no safety certification, no human-in-the-loop testing.

## Future work

- Replace the MG995s with closed-loop actuators
- Instance segmentation for small or reflective instruments
- Dynamic collision avoidance (for example, hands and draped equipment)
- Force and slip sensing in the gripper
- A sterile-compatible design, including drape mounting and cleanable surfaces

## Repository contents

```
SURGBOT/
├── README.md
├── images/            # photos, plots, wiring and power board
├── trialrun1.mp4      # test run video
├── src/
│   ├── speech/        # speech recognition and command parsing
│   ├── vision/        # detection and coordinate conversion
│   ├── kinematics/    # FK, IK, trajectory generation
│   └── arduino/       # servo control sketch
└── models/            # trained weights or link to download
```
<!-- The repository currently only has the README, two photos and the video. Add the code and adjust this tree to match -->

## Getting started

<!-- Fill in once the code is uploaded: -->
<!-- 1. Hardware wiring (servo pins, serial port) -->
<!-- 2. Raspberry Pi setup and dependencies (Python version, OpenCV, speech library, detector framework) -->
<!-- 3. Camera calibration steps -->
<!-- 4. Flashing the Arduino sketch -->
<!-- 5. Running the main script -->

## Report

<!-- Link to the project report PDF if you want to include it in the repo -->

## License

<!-- e.g. MIT -->

## Acknowledgements

Dr. Amritanshu Gupta, Minor Specialization Project Coordinator, Manipal Institute of Technology.
