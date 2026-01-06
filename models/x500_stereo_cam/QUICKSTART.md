# Quick Start Guide - x500_stereo_cam with VINS-Fusion

## Prerequisites

1. **PX4 Autopilot** (already installed)
2. **Gazebo Sim** (Harmonic or Garden)
3. **ROS2 Humble**
4. **ros_gz_bridge** package
5. **VINS-Fusion** (optional, for VIO)

## Step 1: Test the Model in Gazebo

```bash
cd ~/workspace/pnk-avoidance/px4

# Build and launch
make px4_sitl gz_x500_stereo_cam
```

You should see:
- Gazebo window with x500 quadrotor
- Stereo camera mounted on the front
- Two camera views (left and right) if visualize is enabled

## Step 2: Verify Camera Topics

In a new terminal:

```bash
# List all topics
gz topic -l | grep x500_stereo_cam

# You should see:
# /model/x500_stereo_cam/left/image_raw
# /model/x500_stereo_cam/left/camera_info
# /model/x500_stereo_cam/right/image_raw
# /model/x500_stereo_cam/right/camera_info
# /model/x500_stereo_cam/stereo_imu

# Check image publishing rate
gz topic -i -t /model/x500_stereo_cam/left/image_raw
```

## Step 3: Install ROS2-Gazebo Bridge

```bash
# For Gazebo Harmonic
sudo apt install ros-humble-ros-gzharmonic-ros-gz-bridge

# For Gazebo Garden
sudo apt install ros-humble-ros-gzgarden-ros-gz-bridge
```

## Step 4: Bridge Topics to ROS2

In a new terminal:

```bash
cd ~/workspace/pnk-avoidance/px4/Tools/simulation/gz/models/x500_stereo_cam

# Source ROS2
source /opt/ros/humble/setup.bash

# Launch bridge
ros2 run ros_gz_bridge parameter_bridge --ros-args \
  -p config_file:=$(pwd)/stereo_vins_bridge.yaml
```

## Step 5: Verify ROS2 Topics

In a new terminal:

```bash
source /opt/ros/humble/setup.bash

# List ROS2 topics
ros2 topic list | grep cam

# Check camera publishing rate
ros2 topic hz /cam0/image_raw
ros2 topic hz /cam1/image_raw
ros2 topic hz /imu0

# View images in RViz2
rviz2
# Add Image display, set topic to /cam0/image_raw
```

## Step 6 (Optional): Run VINS-Fusion

### Install VINS-Fusion

```bash
# Clone VINS-Fusion
cd ~/
mkdir -p vins_ws/src
cd vins_ws/src
git clone https://github.com/HKUST-Aerial-Robotics/VINS-Fusion.git

# Build
cd ~/vins_ws
catkin_make
source devel/setup.bash
```

### Run VINS-Fusion

Terminal 1: PX4 + Gazebo (already running)
Terminal 2: ROS2 Bridge (already running)

Terminal 3: ROS1 Bridge (VINS is ROS1, our topics are ROS2)
```bash
# Install ROS1-ROS2 bridge if not already installed
sudo apt install ros-humble-ros1-bridge

# Run bridge
source /opt/ros/humble/setup.bash
ros2 run ros1_bridge dynamic_bridge --bridge-all-topics
```

Terminal 4: Run VINS-Fusion
```bash
source ~/vins_ws/devel/setup.bash

# Copy the config file to VINS config directory
cp ~/workspace/pnk-avoidance/px4/Tools/simulation/gz/models/x500_stereo_cam/vins_fusion_config.yaml \
   ~/vins_ws/src/VINS-Fusion/config/stereo_gazebo/

# Run VINS
roslaunch vins vins_rviz.launch \
  config_path:=~/vins_ws/src/VINS-Fusion/config/stereo_gazebo/vins_fusion_config.yaml
```

## Step 7 (Optional): Send VINS Output to PX4

If you want to feed VINS odometry back to PX4 for autonomous flight:

### Enable EKF2 Vision Fusion

Edit the airframe file or set parameters manually:

```bash
# In PX4 console
param set EKF2_EV_CTRL 15      # Enable position + velocity + yaw fusion
param set EKF2_HGT_REF 3       # Use vision for altitude (optional)
param set EKF2_EV_POS_X 0.12   # Camera X position
param set EKF2_EV_POS_Y 0.0    # Camera Y position
param set EKF2_EV_POS_Z 0.05   # Camera Z position
param save
```

### Bridge VINS Output to PX4

Create a simple Python script to republish VINS odometry to PX4:

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import PoseStamped
from px4_msgs.msg import VehicleOdometry

class VinsToVisionPose(Node):
    def __init__(self):
        super().__init__('vins_to_vision_pose')
        self.subscription = self.create_subscription(
            PoseStamped,
            '/vins_estimator/camera_pose',
            self.vins_callback,
            10)
        self.publisher = self.create_publisher(
            VehicleOdometry,
            '/fmu/in/vehicle_visual_odometry',
            10)

    def vins_callback(self, msg):
        vision_msg = VehicleOdometry()
        vision_msg.timestamp = self.get_clock().now().nanoseconds // 1000
        vision_msg.pose_frame = VehicleOdometry.POSE_FRAME_NED

        # Convert ENU to NED
        vision_msg.position[0] = msg.pose.position.y   # North = East
        vision_msg.position[1] = msg.pose.position.x   # East = North
        vision_msg.position[2] = -msg.pose.position.z  # Down = -Up

        # Convert orientation ENU to NED
        vision_msg.q[0] = msg.pose.orientation.w
        vision_msg.q[1] = msg.pose.orientation.y
        vision_msg.q[2] = msg.pose.orientation.x
        vision_msg.q[3] = -msg.pose.orientation.z

        self.publisher.publish(vision_msg)

def main():
    rclpy.init()
    node = VinsToVisionPose()
    rclpy.spin(node)
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

## Troubleshooting

### Camera images not showing
- Check Gazebo is running: `gz topic -l`
- Verify bridge config path is correct
- Ensure ROS2 is sourced: `source /opt/ros/humble/setup.bash`

### VINS not detecting features
- Make sure Gazebo world has textured surfaces
- Check camera topics are synchronized
- Verify IMU is publishing: `ros2 topic hz /imu0`

### Low frame rate
- Reduce image resolution in model.sdf
- Increase simulation real-time factor
- Check system resources (CPU/GPU usage)

## Camera Specifications Summary

| Parameter | Value |
|-----------|-------|
| Baseline | 120mm |
| Resolution | 752x480 |
| Frame Rate | 20 Hz |
| Format | Grayscale (L8) |
| FOV | 80° |
| IMU Rate | 200 Hz |

## File Locations

- **Model**: `Tools/simulation/gz/models/x500_stereo_cam/`
- **Airframe**: `ROMFS/px4fmu_common/init.d-posix/airframes/4030_gz_x500_stereo_cam`
- **Bridge Config**: `Tools/simulation/gz/models/x500_stereo_cam/stereo_vins_bridge.yaml`
- **VINS Config**: `Tools/simulation/gz/models/x500_stereo_cam/vins_fusion_config.yaml`

## Next Steps

1. Test in different Gazebo worlds with various textures
2. Tune VINS parameters for optimal performance
3. Implement collision avoidance using VINS odometry
4. Create autonomous flight missions using vision-based navigation

## Support

For issues or questions:
- PX4 Forum: https://discuss.px4.io/
- PX4 Slack: https://px4.slack.com/
- VINS-Fusion Issues: https://github.com/HKUST-Aerial-Robotics/VINS-Fusion/issues
