# 06. Troubleshooting

[← Automatic Bringup](05-automatic-bringup.md) · [Back to README](../README.md)

Use this guide after checking that the robot is in a safe state. Do not repeatedly send motion commands while hardware connectivity is uncertain.

## Fast Diagnostic Set

On the Jetson, with the same environment used for bringup:

```bash
printenv ROS_DOMAIN_ID TURTLEBOT3_MODEL LDS_MODEL
ls -l /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
ros2 node list
ros2 topic list
```

On the laptop:

```bash
echo "$ROS_DOMAIN_ID"
ping -c 4 <jetson-ip>
ros2 topic list
```

## Laptop Cannot Find Jetson ROS 2 Topics

| Diagnosis | Resolution |
| --- | --- |
| Different network or no IP reachability | In the tested setup, enable the laptop hotspot and connect the Jetson to it. Router Wi-Fi and an external hotspot also work when both devices are on the same network and client isolation is disabled. Run `hostname -I` on the Jetson to obtain its current address, then verify `ping`. |
| Domain mismatch | Set `export ROS_DOMAIN_ID=30` in the active terminal on both computers, then restart the relevant ROS 2 commands. |
| Bringup not running | Start bringup on the Jetson and run `ros2 topic list` locally first. |
| Discovery still fails | Run `ros2 doctor --report` on both machines and record network/firewall findings. Hostname, IP, DDS implementation, and firewall configuration are TBD; do not change them without recording the tested change. |

## Bringup Fails to Start

```bash
source /opt/ros/humble/setup.bash
source ~/turtlebot3_ws/install/setup.bash
export ROS_DOMAIN_ID=30
export TURTLEBOT3_MODEL=burger
export LDS_MODEL=LDS-02
ros2 launch turtlebot3_bringup robot.launch.py
```

If the command reports a missing package, confirm the workspace overlay exists and has been sourced:

```bash
ros2 pkg prefix turtlebot3_bringup
ros2 pkg prefix turtlebot3_node
```

If `LDS_MODEL` is unset or misspelled, the launch file may select the wrong driver. Verify it exactly with `printenv LDS_MODEL`.

## OpenCR Is Not Detected

1. Inspect serial devices:

   ```bash
   ls -l /dev/ttyACM* 2>/dev/null
   sudo dmesg | tail -n 30
   ```

2. Reseat the known-good USB cable and verify that OpenCR is powered.
3. If a confirmed OpenCR path is not `/dev/ttyACM0`, launch with it explicitly:

   ```bash
   ros2 launch turtlebot3_bringup robot.launch.py usb_port:=<opencr-device-path>
   ```

4. If no serial device appears, stop and inspect physical power/cabling; do not assume a software-only fault.

## LDS-02 Is Not Detected

1. Verify the required variable and serial device:

   ```bash
   echo "$LDS_MODEL"
   ls -l /dev/ttyUSB* 2>/dev/null
   sudo dmesg | tail -n 30
   ```

2. The verified launch file expects `LDS_MODEL=LDS-02` and `/dev/ttyUSB0`.
3. Reseat the LiDAR USB connection and restart bringup after the device appears.
4. If the device appears under another path, record it. The current launch file fixes the LDS-02 port at `/dev/ttyUSB0`; inspect/update the launch configuration rather than applying an undocumented assumption.

## Robot Does Not Move

| Check | Command or action | Healthy result |
| --- | --- | --- |
| Teleop is publishing | `ros2 topic echo /cmd_vel` while pressing a motion key | Twist messages arrive. |
| Driver is subscribing | `ros2 topic info /cmd_vel -v` | At least one robot-side subscriber is present. |
| OpenCR is available | `ls -l /dev/ttyACM*` and review bringup terminal | OpenCR path exists; no repeated connection error. |
| Bringup and environment | Run `source ~/.bashrc`, set `LDS_MODEL=LDS-02`, then start bringup | The verified software sequence is active; no separate motor-enable command has been recorded. |
| Safety | Press `s` or space, then inspect obstacle/physical condition | Robot remains safe before another test. |

## `/cmd_vel` Is Not Received

On the laptop, verify publishing:

```bash
ros2 topic echo /cmd_vel
```

On the Jetson, verify the driver endpoint:

```bash
ros2 topic info /cmd_vel -v
```

If the laptop sees commands but the Jetson does not list `/cmd_vel`, return to the network and `ROS_DOMAIN_ID=30` checks. If the Jetson sees `/cmd_vel` but no subscriber is listed, bringup or the TurtleBot3 driver is not healthy; review its terminal output and OpenCR detection.

## Motor Power Is Not Active

The verified power chain is **USB-C powerbank → PD decoy configured for 12 V → Jetson and OpenCR**. The project confirms that teleoperation has worked. The verified software sequence is `source ~/.bashrc`, `export LDS_MODEL=LDS-02`, and then start bringup. No separate physical motor-enable step has been recorded.

If the robot does not move, first check that the powerbank is charged and connected, the PD decoy remains configured for 12 V, and both Jetson and OpenCR are powered. Then check OpenCR detection, `/cmd_vel` publication, and the TurtleBot3 driver subscriber before changing configuration. The powerbank model and available output current are TBD.

## When to Collect More Information

Record the command, full error text, `ros2 node list`, `ros2 topic list`, and `dmesg` tail before changing configuration. This makes the issue reproducible and avoids replacing verified settings with generic defaults.
