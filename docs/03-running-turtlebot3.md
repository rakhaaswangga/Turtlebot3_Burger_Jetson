# 03. Running TurtleBot3

[← System Configuration](02-system-configuration.md) · [Next: Teleoperation →](04-teleoperation.md)

Follow this procedure for a supervised demonstration. Keep the robot on a clear, level area with its wheels free to move. Be ready to stop it immediately.

## What Bringup Does

The verified command starts `turtlebot3_bringup/robot.launch.py`. It launches:

| Component | Role |
| --- | --- |
| `turtlebot3_ros` (`turtlebot3_node`) | Connects to OpenCR, receives velocity commands, and publishes robot-related state including odometry/TF. |
| `robot_state_publisher` | Publishes the robot link transforms from the TurtleBot3 description. |
| LDS-02 driver | Publishes laser scan data from the LDS-02. |

Exact node names can vary with ROS launch naming; use `ros2 node list` as the source of truth during the demonstration.

## 1. Power the Robot and Jetson

1. Confirm the physical connections: OpenCR is connected to Jetson USB and LDS-02 is connected to Jetson USB.
2. Connect the USB-C output of the powerbank to the PD decoy module.
3. Confirm the PD decoy is configured to supply **12 V** to the Jetson and OpenCR, then power the system using the established hardware procedure.
4. Wait for the Jetson to finish booting, then log in locally or remotely.

The verified power chain is **USB-C powerbank → PD decoy (12 V) → Jetson and OpenCR**. The powerbank model and its output-current capability are **TBD**. No separate physical motor-enable procedure has been recorded: in the verified software sequence, the environment is loaded with `source ~/.bashrc` and then bringup is started. Always verify that the robot is safe before commanding movement.

## 2. Join the Same Network

The tested arrangement is to enable the laptop hotspot and connect the Jetson to it. Router Wi-Fi and an external hotspot are alternatives; see [System Configuration](02-system-configuration.md#network-configuration). On the Jetson, find its current dynamic address with `hostname -I`. Then, on the laptop, confirm it can be reached:

```bash
ping -c 4 <jetson-ip>
```

Expected result: replies from the Jetson. Replace the placeholder with the address currently assigned by the active network.

## 3. Check the Jetson Environment

On the Jetson terminal:

```bash
source ~/.bashrc
export LDS_MODEL=LDS-02
printenv ROS_DISTRO ROS_DOMAIN_ID TURTLEBOT3_MODEL LDS_MODEL
ls -l /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
```

Expected environment values: `humble`, `30`, `burger`, and `LDS-02`. If the model or domain is empty, set it with `export ROS_DOMAIN_ID=30` and `export TURTLEBOT3_MODEL=burger`, then re-check. The verified serial defaults are OpenCR at `/dev/ttyACM0` and LDS-02 at `/dev/ttyUSB0`.

## 4. Start Bringup on the Jetson

```bash
ros2 launch turtlebot3_bringup robot.launch.py
```

If OpenCR appears at a different confirmed device path, use:

```bash
ros2 launch turtlebot3_bringup robot.launch.py usb_port:=<opencr-device-path>
```

Keep this terminal open. Successful startup should show launch output without repeated hardware connection errors.

## 5. Inspect Active Nodes

Open a second terminal on the Jetson, load the same environment, and run:

```bash
ros2 node list
```

Expected result: entries corresponding to the TurtleBot3 driver, state publisher, and LDS-02 driver. Names are implementation-dependent; do not treat a fixed name as a requirement.

## 6. Inspect ROS 2 Topics

```bash
ros2 topic list
```

Expected result: the graph includes command velocity, laser scan, odometry, and TF-related topics. Inspect the actual types and endpoints instead of guessing:

```bash
ros2 topic info /cmd_vel -v
ros2 topic info /scan -v
ros2 topic info /odom -v
```

`/cmd_vel` should have a TurtleBot3 driver subscriber when bringup is healthy.

## 7. Confirm LiDAR and Odometry Data

In separate Jetson terminals:

```bash
ros2 topic echo /scan --once
ros2 topic echo /odom --once
```

Expected result: `/scan` returns a `sensor_msgs/msg/LaserScan` message, and `/odom` returns a `nav_msgs/msg/Odometry` message. Odometry may remain stationary until the robot moves; it should still be published.

If either topic is absent, see [Troubleshooting](06-troubleshooting.md) before proceeding to [Teleoperation](04-teleoperation.md).

## 8. Stop Bringup Safely

1. Stop teleoperation first and ensure no key is being held.
2. In the terminal running bringup, press `Ctrl+C` once.
3. Wait for the launch process to exit before removing power or USB cables.
4. Turn off the robot using the established hardware procedure.

Never disconnect OpenCR or LiDAR while the stack is actively using it unless emergency recovery requires it.
