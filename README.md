# TurtleBot3 Burger Jetson Orin Nano Guide

Technical guide and operation manual for a TurtleBot3 Burger migrated from a Raspberry Pi 3 to an NVIDIA Jetson Orin Nano. It is intended for project demonstrations: understanding the system, starting the robot-side stack, and teleoperating the robot from a laptop.

> Scope: this repository documents the configuration that has been tested. It does not claim SLAM, Navigation2, or autonomous navigation are operational.

## Objectives

- Record the Jetson migration and the verified robot configuration.
- Explain communication between the Jetson, OpenCR, LDS-02, and laptop.
- Provide a repeatable bringup and keyboard-teleoperation procedure.
- Provide focused checks for common demonstration issues.

## System at a Glance

| Area | Verified configuration |
| --- | --- |
| Robot | TurtleBot3 Burger |
| Robot-side computer | NVIDIA Jetson Orin Nano |
| Original computer | Raspberry Pi 3 (replaced) |
| Motor controller | OpenCR 1.0 |
| LiDAR | LDS-02 |
| Robot and laptop OS | Ubuntu 22.04.5 |
| ROS distribution | ROS 2 Humble |
| ROS domain | `30` |
| TurtleBot model | `burger` |
| Network | Laptop hotspot (used in testing) or shared Wi-Fi |
| Power chain | USB-C powerbank → PD decoy set to 12 V → Jetson and OpenCR |
| Powerbank model | TBD |

```mermaid
flowchart LR
  Laptop[Laptop\nTeleoperation] <-->|Wi-Fi / hotspot\nROS 2, domain 30| Jetson[Jetson Orin Nano\nTurtleBot3 bringup]
  Jetson <-->|USB serial| OpenCR[OpenCR 1.0]
  Jetson -->|USB serial| Lidar[LDS-02]
  OpenCR --> Motors[Wheel motors]
```

## General System Workflow

This operational flow complements the communication architecture above. It shows the normal sequence from power-on to teleoperation, not the physical communication links.

```mermaid
flowchart TD
  Power[1. Power on robot and Jetson] --> Boot[2. Jetson boots and prepares ROS 2 environment]
  Boot --> Bringup[3. Start TurtleBot3 bringup\nmanual or automatic]
  Bringup --> Devices[4. Initialize OpenCR and LDS-02]
  Devices --> ROS[5. ROS 2 nodes and sensors run]
  ROS --> Network[6. Connect laptop to the same network]
  Network --> Teleop[7. Laptop teleoperation publishes /cmd_vel]
  Teleop --> Motion[Robot moves]
  ROS --> Feedback[8. Robot publishes sensor data and status through ROS 2]
```

## Current Achievement

- TurtleBot3 bringup runs on the Jetson.
- OpenCR and LDS-02 have been configured and run successfully.
- ROS 2 communication between laptop and Jetson has been tested.
- Keyboard teleoperation from the laptop moves the robot.

## Documentation

1. [System overview](docs/01-system-overview.md)
2. [System configuration](docs/02-system-configuration.md)
3. [Running TurtleBot3](docs/03-running-turtlebot3.md)
4. [Keyboard teleoperation](docs/04-teleoperation.md)
5. [Automatic bringup](docs/05-automatic-bringup.md)
6. [Troubleshooting](docs/06-troubleshooting.md)

## License

This project is licensed under the [MIT License](LICENSE).

## Quick Start

On the Jetson, load the saved ROS environment and start the robot stack:

```bash
source ~/.bashrc
export LDS_MODEL=LDS-02
ros2 launch turtlebot3_bringup robot.launch.py
```

On a laptop connected to the same network, use the same ROS domain and run teleoperation:

```bash
source /opt/ros/humble/setup.bash
export ROS_DOMAIN_ID=30
export TURTLEBOT3_MODEL=burger
ros2 run turtlebot3_teleop teleop_keyboard
```

Ensure that `turtlebot3_teleop` is installed in the laptop ROS 2 environment, or source the local TurtleBot3 workspace before running the command:

```bash
source ~/turtlebot3_ws/install/setup.bash
```

See [System Configuration](docs/02-system-configuration.md) for laptop-hotspot, router, external-hotspot, and SSH communication procedures. See [Running TurtleBot3](docs/03-running-turtlebot3.md) before driving the robot.
