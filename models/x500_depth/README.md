# x500_depth - Depth Camera Model for Point Cloud Generation

## Overview

This model provides an x500 quadrotor equipped with an OakD-Lite depth camera system for point cloud generation and obstacle avoidance applications.

## Specifications

### OakD-Lite Depth Camera

#### Depth Sensor (StereoOV7251)
- **Resolution**: 640 x 480 pixels
- **Format**: R_FLOAT32 (32-bit float depth values)
- **Field of View**: 73 degrees (1.274 rad)
- **Range**: 0.2m - 19.1m
- **Frame Rate**: 30 Hz
- **Topics**:
  - Depth image: `/depth_camera`
  - Point cloud: `/depth_camera/points`
  - Camera info: `/depth_camera/camera_info`

#### RGB Sensor (IMX214)
- **Resolution**: 1920 x 1080 pixels
- **Field of View**: 69 degrees (1.204 rad)
- **Frame Rate**: 30 Hz
- **Topics**:
  - RGB image: `/IMX214/image`
  - Camera info: `/IMX214/camera_info`

### Camera Mounting Position
Relative to vehicle center of gravity:
- **X (forward)**: 0.12 m
- **Y (left)**: 0.03 m
- **Z (up)**: 0.242 m

## Usage

### Launch Simulation

```bash
cd ~/workspace/pnk-avoidance/px4
make px4_sitl gz_x500_depth
```

### Check Available Topics

```bash
# List all Gazebo topics
gz topic -l | grep -E "(depth_camera|IMX214)"

# Echo point cloud data
gz topic -e -t /depth_camera/points

# Echo depth image
gz topic -e -t /depth_camera
```

## ROS2 Integration

### 1. Install ROS2-Gazebo Bridge

```bash
# For Gazebo Harmonic + ROS2 Humble
sudo apt install ros-humble-ros-gzharmonic

# For Gazebo Garden + ROS2 Humble
sudo apt install ros-humble-ros-gzgarden
```

### 2. Launch Bridge

```bash
# Navigate to model directory
cd ~/workspace/pnk-avoidance/px4/Tools/simulation/gz/models/x500_depth

# Source ROS2
source /opt/ros/humble/setup.bash

# Launch bridge with config file
ros2 run ros_gz_bridge parameter_bridge --ros-args \
  -p config_file:=$(pwd)/depth_bridge.yaml
```

### 3. Verify ROS2 Topics

```bash
# List available topics
ros2 topic list | grep camera

# Check point cloud publishing rate
ros2 topic hz /camera/depth/points

# Visualize in RViz2
rviz2 -d your_config.rviz
```

## ROS2 Topic Mapping

| Gazebo Topic | ROS2 Topic | Message Type |
|--------------|------------|--------------|
| `/depth_camera/points` | `/camera/depth/points` | `sensor_msgs/PointCloud2` |
| `/depth_camera` | `/camera/depth/image_raw` | `sensor_msgs/Image` |
| `/depth_camera/camera_info` | `/camera/depth/camera_info` | `sensor_msgs/CameraInfo` |
| `/IMX214/image` | `/camera/color/image_raw` | `sensor_msgs/Image` |
| `/IMX214/camera_info` | `/camera/color/camera_info` | `sensor_msgs/CameraInfo` |
| `/clock` | `/clock` | `rosgraph_msgs/Clock` |

## Complete Workflow

### Terminal 1: Launch PX4 + Gazebo
```bash
cd ~/workspace/pnk-avoidance/px4
make px4_sitl gz_x500_depth
```

### Terminal 2: Bridge to ROS2
```bash
source /opt/ros/humble/setup.bash
cd ~/workspace/pnk-avoidance/px4/Tools/simulation/gz/models/x500_depth
ros2 run ros_gz_bridge parameter_bridge --ros-args \
  -p config_file:=$(pwd)/depth_bridge.yaml
```

### Terminal 3: Visualize Point Cloud
```bash
source /opt/ros/humble/setup.bash
ros2 run rviz2 rviz2
# Add PointCloud2 display and set topic to /camera/depth/points
```

### Terminal 4: Fix camera link at the origin of map and will never move it, quick to see the pointcloud
```bash
ros2 run tf2_ros static_transform_publisher 0 0 0 0 0 0 map camera_link
```

## Using Point Cloud for Obstacle Avoidance

### With PX4 Avoidance

```bash
# Run local planner (example)
ros2 launch local_planner local_planner_depth_camera.launch.py
```

### With octomap_server

```bash
# Install octomap
sudo apt install ros-humble-octomap-server

# Run octomap to generate 3D occupancy grid
ros2 run octomap_server octomap_server_node --ros-args \
  -p frame_id:=camera_link \
  -r cloud_in:=/camera/depth/points
```

## Troubleshooting

### No Point Cloud in ROS2
- Verify Gazebo is publishing: `gz topic -e -t /depth_camera/points`
- Check bridge is running: `ros2 topic list | grep depth`
- Ensure simulation is not paused

### Point Cloud Has Wrong Frame
- Check TF tree: `ros2 run tf2_tools view_frames`
- May need to add static transform publisher for camera frame

### Bridge Connection Issues
- Ensure `GZ_IP` and `ROS_DOMAIN_ID` are set correctly if using distributed setup
- Check for firewall blocking Gazebo transport ports

### Fix the rviz2: Message Filter dropping message
- `ros2 run tf2_ros static_transform_publisher 0 0 0 0 0 0 map camera_link`
- Quick fix static transform publisher instead of write urdf file.

## Model Files

```
x500_depth/
├── model.sdf           # Gazebo model definition
├── model.config        # Model metadata
├── depth_bridge.yaml   # ROS2-Gazebo bridge config
├── README.md           # This file
├── LICENSE             # License file
└── thumbnails/         # Model preview images
```

## Related Files

- Airframe: `ROMFS/px4fmu_common/init.d-posix/airframes/4002_gz_x500_depth`
- OakD-Lite model: `Tools/simulation/gz/models/OakD-Lite/`

## References

- Gazebo Depth Camera: https://gazebosim.org/api/sensors/8/depth_camera_tutorial.html
- ROS2 Gazebo Bridge: https://github.com/gazebosim/ros_gz
- PX4 Computer Vision: https://docs.px4.io/main/en/computer_vision/
- OakD-Lite specs: https://docs.luxonis.com/projects/hardware/en/latest/pages/DM9095.html
