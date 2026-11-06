# 02. System Configuration

[← System Overview](01-system-overview.md) · [Next: Running TurtleBot3 →](03-running-turtlebot3.md)

This page records the configuration used by the working system. It is not an installation-from-scratch guide.

## Role Assignment

| System | Role | Verified software |
| --- | --- | --- |
| Jetson Orin Nano | Robot-side computer; runs bringup and hardware drivers | Ubuntu 22.04.5, ROS 2 Humble |
| Laptop | Teleoperation workstation | Ubuntu 22.04.5, ROS 2 Humble |

## Required ROS Environment

On the verified Jetson account, `~/.bashrc` loads the ROS 2 Humble setup, the TurtleBot3 workspace, `ROS_DOMAIN_ID=30`, and `TURTLEBOT3_MODEL=burger`. Start each new Jetson terminal with:

```bash
source ~/.bashrc
```

Then select the attached LiDAR before bringup:

```bash
source ~/turtlebot3_ws/install/setup.bash
export LDS_MODEL=LDS-02
```

`robot.launch.py` reads `TURTLEBOT3_MODEL` and `LDS_MODEL`. With `LDS_MODEL=LDS-02`, the verified launch configuration selects the `ld08_driver` launch file. The values can be made persistent in a shell configuration file only after they have been confirmed for the intended user account.

Verify the active shell:

```bash
printenv ROS_DISTRO ROS_DOMAIN_ID TURTLEBOT3_MODEL LDS_MODEL
```

Expected values on the Jetson include `humble`, `30`, `burger`, and `LDS-02`.

If `ROS_DOMAIN_ID` or `TURTLEBOT3_MODEL` is empty after `source ~/.bashrc`, set it only for the current terminal and re-check:

```bash
export ROS_DOMAIN_ID=30
export TURTLEBOT3_MODEL=burger
export LDS_MODEL=LDS-02
printenv ROS_DOMAIN_ID TURTLEBOT3_MODEL LDS_MODEL
```

Do not add persistent exports until the account and workspace path have been confirmed.

## Network Configuration

The tested arrangement is a **hotspot created by the laptop**, with the Jetson connected as a client. A router Wi-Fi network and an external hotspot are supported alternatives. In every arrangement, the laptop and Jetson must join the same IP network.

The Jetson IP address is expected to change when the hotspot or network changes. Do not store a fixed address in commands. Find the current address after connecting:

Find the local addresses on either machine:

```bash
hostname
hostname -I
ip -br address
```

Use the current Jetson address in the next commands. The placeholder `<jetson-ip>` always means the address discovered for the active network.

### Option A — Laptop Hotspot (Tested)

1. On the **laptop** (Ubuntu 22.04.5), open **Settings → Wi-Fi**.
2. Open the Wi-Fi menu (three dots) and choose **Turn On Wi-Fi Hotspot…**.
3. Set a hotspot name and password, then enable it. Do not publish the password in this repository.
4. On the **Jetson**, open its Wi-Fi network menu, select the laptop hotspot name, and enter the password.
5. On the Jetson, run `hostname -I` to obtain its current address.
6. On the laptop, verify reachability and SSH as described below.

The exact menu labels can vary slightly with the desktop environment. This option is verified for the project; its SSID and password are intentionally not documented.

### Option B — Router Wi-Fi

1. Connect the laptop and Jetson to the **same router SSID**.
2. On the Jetson, run `hostname -I` to obtain the address assigned by the router.
3. From the laptop, use the ping and SSH checks below.

This is a standard alternative, not a separately recorded project test. Client isolation on some guest networks can block peer-to-peer communication; use a normal local network if ping fails.

### Option C — External Hotspot

1. Connect both the laptop and Jetson to the **same external hotspot SSID**.
2. Obtain the current Jetson address with `hostname -I`.
3. Use the ping and SSH checks below.

This is also a standard alternative, not a separately recorded project test. Some hotspots isolate clients; if the laptop cannot ping the Jetson, use the laptop hotspot or a router network that permits local clients.

### Ping and SSH

From the laptop, test reachability using the current Jetson address:

```bash
ping -c 4 <jetson-ip>
```

Expected result: four replies with no packet loss. IP reachability is necessary, but ROS 2 discovery must also be checked with `ros2 topic list` after bringup starts.

To open a Jetson terminal remotely from the laptop:

```bash
ssh <jetson-username>@<jetson-ip>
```

On the first connection, confirm the host-key fingerprint only after verifying it is the intended Jetson, then enter the Jetson user password. The username is **TBD** and must not be guessed.

If SSH reports `Connection refused`, check on the Jetson:

```bash
sudo systemctl status ssh
```

If the SSH service is installed but inactive, the standard recovery is:

```bash
sudo systemctl enable --now ssh
```

If the service is not installed, install `openssh-server` using the approved system-administration procedure. This SSH installation state has not been recorded for the project.

## ROS 2 Communication

ROS 2 nodes use a DDS domain to separate robot networks. Both the laptop and Jetson must use `ROS_DOMAIN_ID=30`; a mismatch normally prevents discovery of topics and nodes.

Useful checks on both computers:

```bash
echo "$ROS_DOMAIN_ID"
ros2 doctor --report
```

Expected domain output:

```text
30
```

## USB Serial Interfaces

The verified TurtleBot3 bringup launch file uses these defaults:

| Device | Default path | Used by |
| --- | --- | --- |
| OpenCR | `/dev/ttyACM0` | `turtlebot3_node` |
| LDS-02 | `/dev/ttyUSB0` | LDS-02 driver |

Before starting bringup on the Jetson, inspect available serial devices:

```bash
ls -l /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
```

For kernel identification after connecting a device:

```bash
sudo dmesg | tail -n 30
```

If the OpenCR device is not `/dev/ttyACM0`, pass the detected path explicitly to bringup:

```bash
ros2 launch turtlebot3_bringup robot.launch.py usb_port:=<opencr-device-path>
```

The LDS-02 path is currently fixed as `/dev/ttyUSB0` by the verified launch file. If it changes, do not assume a replacement path; inspect the launch configuration and update or override it in a controlled, documented change.

## Configuration Boundary

The following are verified: Ubuntu/ROS versions, domain ID, Burger model, expected USB defaults, laptop hotspot as a communication method, and the Jetson `~/.bashrc` environment. These are still **TBD**: Jetson hostname/username, network SSID/password, powerbank model, USB udev aliases, and the laptop’s persistent ROS setup. See [Troubleshooting](06-troubleshooting.md) when a verification step fails.
