# 04. Keyboard Teleoperation

[← Running TurtleBot3](03-running-turtlebot3.md) · [Next: Automatic Bringup →](05-automatic-bringup.md)

## Before You Drive

Confirm that:

- Bringup is still running on the Jetson; follow [Running TurtleBot3](03-running-turtlebot3.md).
- The Jetson is connected to the laptop hotspot (the tested arrangement), router Wi-Fi, or external hotspot on the same network as the laptop. See [System Configuration](02-system-configuration.md#network-configuration).
- Both machines use `ROS_DOMAIN_ID=30`.
- The robot has a clear area and a supervisor can stop it.

## ROS 2 Topic Model

In ROS 2, a **publisher** sends messages to a named **topic**, and a **subscriber** receives messages from that topic. The teleoperation node publishes velocity messages to `/cmd_vel`; the TurtleBot3 driver subscribes to `/cmd_vel` and sends the resulting motion command to OpenCR.

```mermaid
flowchart LR
  Keyboard[Keyboard] --> Teleop[Teleop Node]
  Teleop -->|publishes geometry_msgs/Twist| Cmd[/cmd_vel topic/]
  Cmd -->|subscribed by| Driver[turtlebot3_node]
  Driver -->|USB serial| OpenCR[OpenCR]
  OpenCR --> Motors[Motors]
```

## 1. Prepare the Laptop Terminal

Open a terminal on the laptop and load ROS 2:

```bash
source /opt/ros/humble/setup.bash
export ROS_DOMAIN_ID=30
echo "$ROS_DOMAIN_ID"
```

Expected output:

```text
30
```

If the laptop has a locally built TurtleBot3 workspace, source it only when needed by the local package installation:

```bash
source ~/turtlebot3_ws/install/setup.bash
```

The actual laptop workspace path is **TBD**. The teleop package must be available in its ROS 2 environment.

## 2. Confirm ROS 2 Communication

With bringup active on the Jetson, run on the laptop:

```bash
ros2 topic list
ros2 topic info /cmd_vel -v
```

Expected result: `/cmd_vel` appears and its detailed info shows a subscriber from the robot side. If the topic is missing, do not drive; follow [Troubleshooting](06-troubleshooting.md).

## 3. Start Keyboard Teleoperation

```bash
ros2 run turtlebot3_teleop teleop_keyboard
```

Click the terminal and use the keys displayed by the program. The verified `teleop_keyboard` implementation uses incremental velocity control:

| Key | Action |
| --- | --- |
| `w` | Increase forward linear velocity |
| `x` | Decrease linear velocity; continue pressing to command reverse |
| `a` | Increase angular velocity (turn left) |
| `d` | Decrease angular velocity (turn right) |
| `s` or space | Force stop: set linear and angular velocity to zero |
| `Ctrl+C` | Exit teleoperation |

Each press of `w`, `x`, `a`, or `d` changes the target speed, rather than commanding one fixed movement step. Start with one or two presses and press `s` or space immediately if motion is unexpected.

## 4. Observe `/cmd_vel` (Optional)

To see the commands in a second laptop terminal:

```bash
source /opt/ros/humble/setup.bash
export ROS_DOMAIN_ID=30
ros2 topic echo /cmd_vel
```

Pressing a movement key should produce `geometry_msgs/msg/Twist` messages. `linear.x` represents forward/reverse velocity and `angular.z` represents turning velocity.

## 5. Stop the Robot

1. Press `s` or space in the teleop terminal to publish zero velocity.
2. Confirm the robot has stopped.
3. Press `Ctrl+C` to exit the teleop node.
4. When the demonstration is complete, stop bringup on the Jetson using `Ctrl+C` as described in [Running TurtleBot3](03-running-turtlebot3.md).

If `/cmd_vel` is visible but the robot does not move, check motor power, OpenCR connectivity, and the `/cmd_vel` subscriber in [Troubleshooting](06-troubleshooting.md).
