# Documentation

Everything here is written for someone who has the repository open and wants to understand or reproduce what we built. It is not marketing material. Where something is unfinished, we say so.

## Start here

| Document | Read it when |
|---|---|
| [architecture.md](architecture.md) | You want to know how the pieces talk to each other and why we split them that way |
| [hardware.md](hardware.md) | You are building the arm, wiring it, or calibrating the camera |
| [camera_server.md](camera_server.md) | You are flashing the ESP32 camera board or writing code that consumes its stream |
| [roadmap.md](roadmap.md) | You want to know what is done, what is next, and what is aspirational |
| [Plan, TODO, Idea, Execution overview](<Plan - TODO - Idea -  Execution overview.md>) | You want the raw working notes we planned from, unedited |

## A note on the state of things

This project was built against a hackathon clock, so the repository is uneven on purpose. The camera node is real firmware you can flash right now. The arm exists as a mechanical assembly. The agent layer is designed and partly written. Several directories hold nothing but a placeholder file because we agreed on the structure before the code existed, and that turned out to be worth doing.

If you are evaluating this, [roadmap.md](roadmap.md) has the honest breakdown of what runs today.

## Conventions we follow

* Distances in the vision and planning layer are centimetres, measured from the arm base, never pixels. Pixels stop existing the moment perception hands off.
* Angles are degrees at the servo interface and radians inside the kinematics solver. The conversion happens in one place so it is easy to audit.
* Anything crossing the network is JSON with an explicit schema. Nothing positional, nothing implicit.
* The language model receives numbers and returns decisions. It never produces a coordinate it was not given. See [architecture.md](architecture.md) for why this rule is not negotiable.

Video explaination : https://drive.google.com/file/d/1SCJc25jkBU7YsOWZCFCaDGiPdMcZFlpp/view
