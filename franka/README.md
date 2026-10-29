# Franka Research 3 — Lab Setup and Operation

[← Back to robot documentation](../README.md)

> **Documentation status:** Working lab guide for review. Adapted from the existing *Franka Research 3 — Lab startup, ROS setup & Cartesian impedance control* operator guide (October 2026). Check the procedure against the actual robot and workstation before treating it as an approved operating procedure.
>
> **Safety:** This is a physical, torque-controlled robot. Keep its entire working area clear, know where the physical stop button is, and do not command motion unless Franka Desk shows that the robot is ready. The Cartesian impedance example does **not** plan collision-free paths.

## Contents

- [Lab configuration](#lab-configuration)
- [1. Start the PC and check real-time access](#1-start-the-pc-and-check-real-time-access)
- [2. Work in the ROS workspace](#2-work-in-the-ros-workspace)
- [3. Power on the robot](#3-power-on-the-robot)
- [4. Connect to Desk in Chrome](#4-connect-to-desk-in-chrome)
- [5. Unlock and lock the arm](#5-unlock-and-lock-the-arm)
- [6. Activate FCI](#6-activate-fci)
- [7. Start the Cartesian impedance example](#7-start-the-cartesian-impedance-example)
- [8. Move the arm in RViz](#8-move-the-arm-in-rviz)
- [9. Stop a run normally](#9-stop-a-run-normally)
- [10. Faults, stopping and recovery](#10-faults-stopping-and-recovery)
- [11. Troubleshooting checks](#11-troubleshooting-checks)
- [12. Shut the robot down safely](#12-shut-the-robot-down-safely)
- [Quick-start checklist](#quick-start-checklist)

## Lab configuration

This guide describes the existing **Franka Research 3** laboratory setup, not a generic Franka installation.

| Component | Lab configuration |
| --- | --- |
| Workstation user | `franka2` |
| Operating system | Ubuntu 20.04 |
| Kernel | **PREEMPT_RT** (lab `rt20` boot option) |
| ROS | **ROS 1 Noetic** |
| Robot address | `172.16.0.2` |
| Robot system version | `4.2.2` |
| Robot integration | `libfranka 0.9.2` + `franka_ros 0.9.1` |
| ROS workspace | `/home/franka2/franka_ws` (`~/franka_ws` when logged in as `franka2`) |
| End effector | Franka Hand (gripper) |

**Unlike the UR5 setup, this Franka workstation must be running the real-time kernel for the FCI controller workflow below.**

## 1. Start the PC and check real-time access

Log into Ubuntu as **`franka2`**. Boot into the lab's **`rt20` / PREEMPT_RT** kernel option, not a generic kernel.

> **📷 IMAGE PLACEHOLDER:** Photo of the workstation's GRUB menu showing the `rt20` kernel selection. Suggested file: `images/01-rt-kernel-grub.jpg`.

Open a terminal and check the kernel:

```bash
uname -a
cat /sys/kernel/realtime
```

The kernel name should indicate PREEMPT_RT (typically containing `-rt20`), and the second command should print **`1`**.

Check the current user's real-time scheduling permissions:

```bash
groups
ulimit -r
ulimit -l
```

Expected for the documented lab setup:

- `groups` contains `realtime`.
- `ulimit -r` prints `99`.
- `ulimit -l` prints `102400` (KiB).

**Stop here if these checks fail.** If the machine booted into a generic kernel, reboot and select the real-time kernel. If real-time priority is unavailable (for example `ulimit -r` is `0`), log out and back in after any group-membership change; ask the lab administrator if the problem persists. Do not attempt FCI motion control with the wrong kernel or privileges.

> **📷 IMAGE PLACEHOLDER:** Terminal showing the successful kernel, `groups` and `ulimit` checks. Suggested file: `images/02-realtime-checks.jpg`.

## 2. Work in the ROS workspace

Use VS Code for routine development. Keep your own packages in the existing `franka_ws` workspace, rather than modifying `/opt/ros/noetic`.

```bash
cd ~/franka_ws
code .
```

> **📷 IMAGE PLACEHOLDER:** VS Code opened in `~/franka_ws`, showing the lab workspace layout. Suggested file: `images/03-franka-workspace.jpg`.

The workstation's `~/.bashrc` should already source ROS Noetic and the workspace. Check that ROS can find the Franka packages:

```bash
rospack find franka_control
rospack find franka_example_controllers
rospack find franka_gripper
```

If they are not available, source the environments explicitly:

```bash
source /opt/ros/noetic/setup.bash
source ~/franka_ws/devel/setup.bash
```

When you change local source code, rebuild the workspace:

```bash
cd ~/franka_ws
catkin_make -DCMAKE_BUILD_TYPE=Release
source devel/setup.bash
```

**Workstation permissions:** The `franka2` account is intentionally not an administrator. Build lab packages within `/home/franka2`. Ask the lab administrator to install system-level dependencies; do not modify `/opt/ros/noetic`, the system Python installation, kernels or other users' files.

## 3. Power on the robot

Before switching on the Franka:

1. Clear people and loose items from the entire robot workspace.
2. Identify the physical stop button and verify that it is accessible and released.
3. Make sure the robot arm and gripper are not obstructed.
4. Turn on the **Franka Control unit using its rear power switch**.
5. Wait until the robot has completely booted before unlocking joints or attempting external control.

> **📷 IMAGE PLACEHOLDER — IMPORTANT:** **Real photo** of the *back of this lab's Franka Control unit*, with the rear power switch clearly identified. Suggested file: `images/04-control-unit-rear-power-switch.jpg`. Do not use a generated picture to identify a safety-relevant control.

> **📷 IMAGE PLACEHOLDER:** Real photo showing the physical stop button and its location in this lab. Suggested file: `images/05-physical-stop-button.jpg`.

**Do not start ROS control during robot boot-up.**

## 4. Connect to Desk in Chrome

On the Franka workstation, open **Google Chrome** and navigate to:

```text
https://172.16.0.2/desk
```

If a local certificate warning appears, follow the lab's IT guidance for the robot's known certificate. Sign into **Franka Desk** using its credentials, which are **separate from the Ubuntu `franka2` login**. A browser session authenticated under another Ubuntu account does not automatically sign in `franka2`.

> **📷 IMAGE PLACEHOLDER:** Chrome showing the Franka Desk sign-in page (without credentials or personal information visible). Suggested file: `images/06-desk-login.jpg`.

Optional connection check:

```bash
ping -c 4 172.16.0.2
```

If Desk or the ping test is inaccessible, check that the robot is powered on and that the dedicated robot network is connected. Do not continue until connectivity is restored.

## 5. Unlock and lock the arm

Before unlocking, make sure nobody is within the robot's reachable workspace or leaning on the arm.

1. In Franka Desk, choose **Unlock joints** (the label may differ slightly by software version).
2. The brakes may click and the arm may settle slightly as they release.
3. Confirm that Desk indicates the arm is unlocked before proceeding.

> **📷 IMAGE PLACEHOLDER:** Desk screen with **Unlock joints** highlighted. Suggested file: `images/07-desk-unlock-joints.jpg`.

Unlocking prepares the arm for guiding or FCI operation; it **does not** start a ROS controller. A locked arm cannot be commanded to move by an external controller.

To immobilise the arm between activities, **stop the ROS controller first**, then select **Lock joints** in Desk. A locked arm is not a substitute for maintaining a clear workspace during active experiments.

> **📷 IMAGE PLACEHOLDER:** Desk screen showing **Lock joints**. Suggested file: `images/08-desk-lock-joints.jpg`.

## 6. Activate FCI

The **Franka Control Interface (FCI)** must be active for ROS control. On the documented robot system version (`4.2.2`), activation is a **separate Desk step**.

1. Confirm the robot is ready and its working area is clear.
2. In Desk, enter **Execution mode** if it is not already selected.
3. Open the Desk **System** menu or sidebar.
4. Select **Activate FCI**.
5. Confirm Desk reports that FCI is active.

> **📷 IMAGE PLACEHOLDER:** Desk **System → Activate FCI** control. Suggested file: `images/09-desk-activate-fci.jpg`.

> **📷 IMAGE PLACEHOLDER:** Desk status indicating **FCI active**. Suggested file: `images/10-desk-fci-active.jpg`.

Do not launch `franka_control` or the example controller until FCI is active. Do not run a competing Desk App and external FCI controller simultaneously.

## 7. Start the Cartesian impedance example

**Preflight:** Real-time kernel and scheduling limits are correct; robot is powered on, unlocked and ready; FCI is active; workspace is clear; no other `franka_control` session is running.

Open a terminal as `franka2`:

```bash
source /opt/ros/noetic/setup.bash
source ~/franka_ws/devel/setup.bash

roslaunch franka_example_controllers cartesian_impedance_example_controller.launch \
  robot_ip:=172.16.0.2 \
  load_gripper:=true
```

This launches the Franka control stack, the Cartesian impedance controller, RViz and an interactive equilibrium-pose marker; it will typically also open `rqt_reconfigure`. `load_gripper:=true` enables the gripper in this example configuration.

**Do not run `franka_control.launch` separately first.** The example launch starts its own control session.

> **📷 IMAGE PLACEHOLDER:** Terminal showing the successful Cartesian impedance launch. Suggested file: `images/11-cartesian-controller-terminal.jpg`.

> **📷 IMAGE PLACEHOLDER:** RViz after the example starts, showing the Franka arm and interactive marker. Suggested file: `images/12-rviz-controller-started.jpg`.

## 8. Move the arm in RViz

> **⚠️ PHYSICAL MOTION WARNING:** Moving the interactive marker commands the **real robot**. Keep the workspace clear, maintain supervision and start with a very small adjustment. This example is **not** MoveIt and performs **no collision-free path planning**.

In RViz, locate the interactive marker near the end effector. It represents a **desired equilibrium pose**, not a MoveIt planning goal.

- Drag an arrow to change the desired XYZ position.
- Rotate a ring to change the desired end-effector orientation.
- After rotation, the marker's translation axes may rotate too, changing subsequent drag directions.
- The arm responds compliantly rather than following rigid position control.

For the initial test, use a small marker movement (the source guide suggests around **10–20 mm**) while directly monitoring the physical arm. Avoid large marker jumps and do not assume obstacle avoidance.

> **📷 IMAGE PLACEHOLDER:** RViz interactive marker showing the translation arrows and rotation rings. Suggested file: `images/13-rviz-interactive-marker.jpg`.

The marker publishes `geometry_msgs/PoseStamped` to:

```text
/cartesian_impedance_example_controller/equilibrium_pose
```

To inspect that topic from another terminal:

```bash
rostopic echo /cartesian_impedance_example_controller/equilibrium_pose
```

### Optional: stiffness tuning

The `rqt_reconfigure` panel may expose:

```text
translational_stiffness
rotational_stiffness
nullspace_stiffness
```

These parameters change the arm's compliant behaviour. **Do not make large changes without understanding the control response.** Use the lab's established settings for routine exercises.

> **📷 IMAGE PLACEHOLDER:** `rqt_reconfigure` panel displaying the three stiffness parameters. Suggested file: `images/14-rqt-stiffness.jpg`.

## 9. Stop a run normally

For routine shutdown of the running example:

1. Focus the terminal running `roslaunch`.
2. Press **Ctrl+C**.
3. Wait until the nodes and the FCI control session have exited before launching another control script.

Do **not** use the physical stop button as the normal way to finish a successful run. It is for hazards and fault recovery as specified by the lab procedure.

> **📷 IMAGE PLACEHOLDER:** Terminal after a normal `Ctrl+C` shutdown, showing the launch ending. Suggested file: `images/15-roslaunch-normal-stop.jpg`.

## 10. Faults, stopping and recovery

**If the robot moves unexpectedly or a person is at risk, immediately press the physical stop button.** Clear the hazard before beginning recovery.

For a software/controller fault **without an immediate hazard**:

1. Stop the current launch with **Ctrl+C**.
2. Read the terminal error and check Desk for fault or recovery messages.
3. If the lab fault procedure requires a stop cycle, press the stop button and wait for the arm to stop or lock.
4. For **this lab's stop button**, release it by **pulling it straight out**; **do not twist it**.
5. Confirm the cause has been corrected, and follow any recovery or acknowledgement prompts in Desk. Releasing the stop button alone may not clear all errors.
6. Only after Desk shows the robot is ready: unlock joints, select Execution mode, reactivate FCI, then restart the ROS launch.

Do not repeatedly stop and release the robot to bypass an unresolved fault. Ask the lab supervisor if the same fault recurs.

> **📷 IMAGE PLACEHOLDER:** Real photo of the lab stop button showing the correct release mechanism. Suggested file: `images/16-stop-button-release.jpg`.

> **📷 IMAGE PLACEHOLDER:** Desk fault/recovery prompt on the lab software version. Suggested file: `images/17-desk-fault-recovery.jpg`.

## 11. Troubleshooting checks

### Real-time scheduling error

Example error:

```text
franka::RealtimeException
libfranka: unable to set realtime scheduling: Operation not permitted
```

Check:

```bash
cat /sys/kernel/realtime
groups
ulimit -r
ulimit -l
```

For this lab, the expected results are: `1`, membership in `realtime`, `99` and `102400`. If group membership has changed, **log out and back in**; just opening another terminal may be insufficient.

### The RViz marker moves but the robot does not

Check which controllers are running:

```bash
rosservice call /controller_manager/list_controllers
```

The `cartesian_impedance_example_controller` should be in state `running`.

Check topic publication and subscribers:

```bash
rostopic info /cartesian_impedance_example_controller/equilibrium_pose
rostopic echo /cartesian_impedance_example_controller/equilibrium_pose
```

If the target changes but the real robot does not respond, stop the launch, inspect Desk's robot/FCI status and check controller and real-time settings. **Do not start a second Franka control stack on top of the first.**

> **📷 IMAGE PLACEHOLDER:** Terminal showing the controller state returned by `/controller_manager/list_controllers`. Suggested file: `images/18-running-controller-check.jpg`.

## 12. Shut the robot down safely

1. Stop the running ROS launch with **Ctrl+C** and wait for it to exit.
2. Return to **Desk** in Chrome.
3. Select **Shut-down / Shutdown robot** (label may differ with software version).
4. Wait for shutdown to finish: robot status lights should go off and the Franka Control cooling fans should stop.
5. **Only after shutdown has finished**, turn off the rear power switch on the Franka Control unit.

**Do not simply switch off rear power while Desk is still shutting down.** For an immediate restart, use the short off period specified by the lab's equipment procedure.

> **📷 IMAGE PLACEHOLDER:** Desk screen showing the **Shutdown robot** control. Suggested file: `images/19-desk-shutdown.jpg`.

> **📷 IMAGE PLACEHOLDER:** Real photo of the rear Control unit switch, clearly showing the off position. Suggested file: `images/20-control-unit-power-off.jpg`.

## Quick-start checklist

Use this only after you are familiar with the full instructions above.

### Preflight

- [ ] Booted into the lab `rt20` / PREEMPT_RT kernel (`cat /sys/kernel/realtime` → `1`).
- [ ] Logged in as `franka2`, with `realtime` group access; `ulimit -r` → `99`, `ulimit -l` → `102400`.
- [ ] Robot workspace is clear and the physical stop button is accessible and released.

### Robot / Desk

- [ ] Turn on the rear Franka Control unit switch and wait for the robot to boot.
- [ ] Open `https://172.16.0.2/desk` in Chrome and sign in.
- [ ] Unlock joints and enter Execution mode.
- [ ] Activate FCI and verify its status.

### Run

```bash
cd ~/franka_ws
roslaunch franka_example_controllers cartesian_impedance_example_controller.launch \
  robot_ip:=172.16.0.2 \
  load_gripper:=true
```

- [ ] Make only a small RViz marker adjustment, watching the physical arm.
- [ ] Finish the run with **Ctrl+C** and wait for the nodes to stop.

### Shutdown

- [ ] In Desk, select **Shutdown robot**.
- [ ] Wait for the status lights and Control cooling fans to stop.
- [ ] Turn off rear power.

## Image notes

The placeholders above are intentionally **text-only** so that the README displays cleanly on GitHub while no photos are available. For example, when you add a real image at `franka/images/04-control-unit-rear-power-switch.jpg`, replace the corresponding placeholder block with:

```markdown
![Rear power switch on the Franka Control unit](images/04-control-unit-rear-power-switch.jpg)
```

Use photos of **your actual lab equipment** for power switches, the stop button and Desk interface controls. The source PDF contained no photographs; no images have been fabricated for this file.

## Further references

These links were included in the original lab guide:

- [Franka Control Interface and real-time setup](https://frankarobotics.github.io/docs/)
- [franka_ros 0.9.1](https://github.com/frankarobotics/franka_ros/tree/0.9.1)
- [libfranka 0.9.2](https://github.com/frankarobotics/libfranka/tree/0.9.2)

---

*Based on the University Robotics Lab's October 2026 Franka Research 3 operator guide. It does not replace the lab's formal safety training or the manufacturer's operating instructions.*