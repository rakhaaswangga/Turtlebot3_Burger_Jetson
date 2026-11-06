# 05. Automatic Bringup

[← Teleoperation](04-teleoperation.md) · [Next: Troubleshooting →](06-troubleshooting.md)

This page describes an optional systemd service that starts TurtleBot3 bringup when the Jetson boots. It complements, rather than replaces, the manual procedure in [Running TurtleBot3](03-running-turtlebot3.md).

## Current Repository Status

The repository `scripts/` directory contains no automatic-bringup script, and no TurtleBot3 systemd unit or desktop autostart configuration was found in this checkout. Therefore, automatic bringup is **not verified as configured or working**.

A supplied reference describes `turtlebot3.service`, but its user and workspace path must not be copied unchanged because they may differ from the target Jetson. The template below uses placeholders and the verified ROS parameters only.

## A. How It Works

`systemd` is the Ubuntu service manager. When `turtlebot3.service` is enabled, systemd starts it during normal boot. The service:

1. waits for the network-online target and the expected USB device units;
2. supplies `ROS_DOMAIN_ID=30`, `TURTLEBOT3_MODEL=burger`, and `LDS_MODEL=LDS-02` as service environment variables;
3. sources ROS 2 Humble and the TurtleBot3 workspace; and
4. runs the existing `ros2 launch turtlebot3_bringup robot.launch.py` command.

The service runs independently of an interactive terminal, so it cannot rely on `~/.bashrc`. Its `ExecStart` explicitly sources the required ROS setup files.

## B. Configure and Enable

### 1. Confirm the Jetson User and Workspace

On the Jetson, identify the account that owns the TurtleBot3 workspace and confirm the overlay exists:

```bash
whoami
ls -l ~/turtlebot3_ws/install/setup.bash
```

The path must exist before installing the service. In the service below, replace `<jetson-user>` with the output of `whoami`. Do not use a user or `/home/...` path copied from another computer.

### 2. Create the Service Unit

Create `/etc/systemd/system/turtlebot3.service` with administrator privileges and use this content:

```ini
[Unit]
Description=TurtleBot3 Burger bringup
Wants=network-online.target
After=network-online.target dev-ttyACM0.device dev-ttyUSB0.device
StartLimitIntervalSec=0

[Service]
Type=simple
User=<jetson-user>
Environment=ROS_DOMAIN_ID=30
Environment=TURTLEBOT3_MODEL=burger
Environment=LDS_MODEL=LDS-02
ExecStartPre=/usr/bin/test -e /dev/ttyACM0
ExecStartPre=/usr/bin/test -e /dev/ttyUSB0
ExecStart=/bin/bash -lc 'source /opt/ros/humble/setup.bash && source /home/<jetson-user>/turtlebot3_ws/install/setup.bash && exec ros2 launch turtlebot3_bringup robot.launch.py usb_port:=/dev/ttyACM0'
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

This template uses the serial-device defaults verified by the existing bringup launch file: OpenCR at `/dev/ttyACM0` and LDS-02 at `/dev/ttyUSB0`. It does not change robot parameters or hardware configuration. The `ExecStartPre` checks cause a failed start to be retried every five seconds if the USB devices are not ready at boot.

### 3. Load and Enable It

Do not run manual bringup while the service is active; both processes would try to use the same serial hardware.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now turtlebot3.service
```

`enable` installs the boot-time link; `--now` starts it immediately. To start or stop it manually:

```bash
sudo systemctl start turtlebot3.service
sudo systemctl stop turtlebot3.service
sudo systemctl restart turtlebot3.service
```

To disable automatic startup and stop the running service:

```bash
sudo systemctl disable --now turtlebot3.service
```

## C. Verification

Run these checks on the Jetson after the service has been enabled or after a reboot.

| Check | Command | Success indicator |
| --- | --- | --- |
| Unit installed | `systemctl cat turtlebot3.service` | Displays the unit content; a missing-unit message means it is not configured. |
| Starts at boot | `systemctl is-enabled turtlebot3.service` | `enabled` |
| Current status | `systemctl status turtlebot3.service --no-pager` | `Active: active (running)` |
| Service log | `journalctl -u turtlebot3.service -b --no-pager` | No recurring launch, OpenCR, or LiDAR connection errors. |
| Bringup process | `pgrep -af 'turtlebot3|ros2 launch'` | A process associated with the TurtleBot3 launch is present. |

For ROS graph checks, use a Jetson terminal with the same ROS environment:

```bash
source ~/.bashrc
export LDS_MODEL=LDS-02
ros2 node list
ros2 topic list
ros2 topic echo /scan --once
ros2 topic echo /odom --once
```

Success indicators are a TurtleBot3 driver, state publisher, and LiDAR-driver node in `ros2 node list`, plus `/scan` and `/odom` in `ros2 topic list`. The two `echo --once` commands should each return one message. Actual node names are implementation-dependent.

## D. Troubleshooting Automatic Bringup

| Symptom | Diagnosis | Action |
| --- | --- | --- |
| `Unit turtlebot3.service could not be found` | The unit was not created at the expected location or systemd was not reloaded. | Verify `/etc/systemd/system/turtlebot3.service`, then run `sudo systemctl daemon-reload`. |
| Service fails immediately | Inspect `systemctl status turtlebot3.service --no-pager` and `journalctl -u turtlebot3.service -b --no-pager`. | Check the `User=` value and that the workspace `install/setup.bash` path exists for that user. |
| ROS command/package not found | The service environment is non-interactive and does not read `~/.bashrc`. | Confirm `ExecStart` sources `/opt/ros/humble/setup.bash` and the correct TurtleBot3 workspace overlay. |
| OpenCR or LDS-02 unavailable at boot | USB enumeration may finish after the service starts, or a serial path may differ. | Check `ls -l /dev/ttyACM* /dev/ttyUSB*` and the service journal. The template retries every five seconds; do not change paths until the actual device is identified. |
| Service repeatedly restarts | A launch or hardware error is causing a non-zero exit. | Follow the first error in `journalctl`; stop the service before running the manual procedure for comparison. |
| Manual bringup cannot open a port | The automatic service already owns the hardware. | Run `sudo systemctl stop turtlebot3.service`, then use the manual procedure. Re-enable/restart the service afterwards if desired. |
| Topics are visible on Jetson but not laptop | Bringup may be healthy while ROS 2 discovery is not. | Use the network and domain checks in [System Configuration](02-system-configuration.md). |

## Direct Hardware Test Still Required

The unit template and commands have been reviewed against the repository’s verified model, ROS domain, launch file, and serial defaults. They have **not** been started or reboot-tested on the Jetson. Test the service while the robot is safely positioned, then verify `/scan`, `/odom`, and teleoperation before using it in a demonstration.
