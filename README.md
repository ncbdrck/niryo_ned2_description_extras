# niryo_ned2_description_extras

Niryo Ned2 sim extras for the `rl_environments` stack. Mirrors
`reactorx200_description`'s role for the RX200:

- Mounts the Ned2 on the **same table model** the RX200 sim uses
  (reactorx200_description's table SDF).
- Adds a **head-mount Kinect v2** at the same pose RX200 uses, so cube
  tracking + extrinsic calibration carry across robots.
- Brings up **ros_control** with the same controller name (`niryo_robot_follow_joint_trajectory_controller`)
  the real Ned2 driver exposes, so RL code paths don't branch between
  sim and real.

Kinect-only for now — ZED2 / D405 variants can be added later if
needed (the YAML extrinsics live in `rl_envs_cube_tracker/config/extrinsics/`).

## Prerequisites

- ROS Noetic
- `niryo_robot_description` (upstream Niryo URDF + meshes)
- `reactorx200_description` (table model)
- `common_sensors` (Kinect v2 xacro)
- `interbotix_xsarm_gazebo` (world file, reused for visual consistency)

`rosdep install --from-paths src --ignore-src -r -y` should resolve
the standard Gazebo + ros_control deps.

## Verify the sim works

```bash
roscore               # terminal 1

# Reach (no gripper):
roslaunch niryo_ned2_description_extras ned2_gazebo.launch                 # terminal 2

# Push / PnP (adaptive gripper attached):
roslaunch niryo_ned2_description_extras ned2_gazebo.launch gripper:=true   # terminal 2
```

Expected:
- Gazebo opens, table at origin, Ned2 standing on top (`base_link` at z=0.78).
- A Kinect v2 model floats in front of the robot looking at the workspace.
- `/joint_states` publishes at 50 Hz (8 joints with `gripper:=true`, 6 without).
- `rostopic list | grep niryo_robot_follow_joint_trajectory_controller`
  shows the arm controller's command + state topics.
- With `gripper:=true`: `rostopic list | grep gazebo_tool_commander` shows
  the mors trajectory controller.
- `/head_mount_kinect2/rgb/image_raw` + `/head_mount_kinect2/depth/image_raw`
  publish camera streams.

## How the gripper variant differs

| | `gripper:=false` (default) | `gripper:=true` |
|---|---|---|
| URDF | `urdf/ned2_kinect.urdf.xacro` | `urdf/ned2_kinect_gripper.urdf.xacro` |
| Wraps | `niryo_ned2_gazebo.urdf.xacro` | `niryo_ned2_gripper1_n_camera.urdf.xacro` |
| Joints in /joint_states | 6 (arm only) | 8 (arm + 2 mors prismatic) |
| Controllers config | `config/ned2_controllers.yaml` | `config/ned2_controllers_w_gripper.yaml` |
| Extra controllers | — | `gazebo_tool_commander` (effort) |
| Wrist camera | yes (Niryo built-in, on `camera_link`, `/gazebo_camera/*`) | yes (same) |
| Suits | Reach | Push, PnP |

The gripper is the **adaptive gripper** (gripper1) — Niryo's standard
accessory. No `custom_gripper` URDF ships with Ned2; designing your
own is described at <https://docs.niryo.com/accessories/grippers/custom-gripper/>
but not needed for the RL stack.

### PID gains note

Niryo's arm transmissions are `EffortJointInterface` — Gazebo needs PID
gains to convert commanded position into torque. Without gains the arm
just sags under gravity and ignores commands (joint state still
publishes but the arm doesn't track trajectories). Both controllers
YAMLs include a `gazebo_ros_control/pid_gains` block with values from
Niryo's own MoveIt sim config (`p: 100, d: 1, i: 1, i_clamp: 1` per arm
joint, same for mors).

## Files

```
niryo_ned2_description_extras/
├── package.xml
├── CMakeLists.txt
├── README.md
├── urdf/
│   ├── ned2.urdf.xacro                # arm only, no kinect
│   ├── ned2_kinect.urdf.xacro         # arm + head-mount kinect2 (reach / push)
│   └── ned2_kinect_gripper.urdf.xacro # arm + kinect + adaptive gripper + built-in wrist camera (pnp)
├── launch/
│   ├── ned2_gazebo.launch       # world + table + ned2 + kinect + control (gripper:=false|true)
│   └── ned2_control.launch      # joint_state + follow_joint_trajectory controllers
└── config/
    ├── ned2_controllers.yaml             # arm-only (6 joints, 25 Hz loop_hz)
    └── ned2_controllers_w_gripper.yaml   # arm + gazebo_tool_commander for mors prismatic joints
```

## Why this exists

Separate description package keeps URDF / meshes / launch files
reusable independent of the RL stack. Standard ROS pattern.

The cube is not spawned by this package — the RL env's reset path
calls `spawn_cube_in_gazebo("red_cube")`, so the world must be
cube-free. `interbotix_xsarm_gazebo/worlds/xsarm_gazebo.world` is
cube-free by default; double-check if you swap worlds.

## Contact

[j.kapukotuwa@research.ait.ie](mailto:j.kapukotuwa@research.ait.ie)
