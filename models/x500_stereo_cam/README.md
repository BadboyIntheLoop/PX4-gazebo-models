# x500_stereo_cam - Stereo Camera Model for VINS-Fusion

## Overview

This model provides an x500 quadrotor equipped with a stereo camera system designed for Visual-Inertial Odometry (VIO) using VINS-Fusion.

## Specifications

### Stereo Camera Configuration
- **Baseline**: 120mm (0.12m) between left and right cameras
- **Resolution**: 752 x 480 pixels (grayscale)
- **Frame Rate**: 20 Hz
- **Field of View**: 80 degrees (1.396 rad)
- **Image Format**: L8 (grayscale, optimized for feature tracking)
- **Topics**:
  - Left: `/model/x500_stereo_cam/left/image_raw`
  - Right: `/model/x500_stereo_cam/right/image_raw`

### Camera Intrinsics (Pre-calibrated)
```
fx = 458.654
fy = 457.296
cx = 367.215
cy = 248.375
```

These values correspond to a focal length suitable for the 80° FOV at 752x480 resolution.

### IMU Sensor
- **Update Rate**: 200 Hz
- **Topic**: `/model/x500_stereo_cam/stereo_imu`
- **Noise**: Configured with realistic gyroscope and accelerometer noise parameters
- **Gyro noise**: 0.009 rad/s (stddev)
- **Accel noise**: 0.017 m/s² (stddev)

### Camera Mounting Position
Relative to vehicle center of gravity:
- **X (forward)**: 0.12 m
- **Y (right)**: 0.0 m
- **Z (down)**: -0.05 m

## Usage

### Launch Simulation

```bash
cd /path/to/PX4-Autopilot
make px4_sitl gz_x500_stereo_cam
```

### Check Available Topics

```bash
# List all Gazebo topics
gz topic -l | grep x500_stereo_cam

# Echo left camera images
gz topic -e -t /model/x500_stereo_cam/left/image_raw

# Echo IMU data
gz topic -e -t /model/x500_stereo_cam/stereo_imu
```

## ROS2 Integration

### 1. Install ROS2-Gazebo Bridge

```bash
# For Gazebo Harmonic
sudo apt install ros-humble-ros-gzharmonic-ros-gz-bridge

# For Gazebo Garden
sudo apt install ros-humble-ros-gzgarden-ros-gz-bridge
```

### 2. Create Bridge Configuration

Save this as `stereo_vins_bridge.yaml`:

```yaml
# Left camera
- topic_name: "/model/x500_stereo_cam/left/image_raw"
  ros_topic_name: "/cam0/image_raw"
  ros_type_name: "sensor_msgs/msg/Image"
  gz_type_name: "gz.msgs.Image"

- topic_name: "/model/x500_stereo_cam/left/camera_info"
  ros_topic_name: "/cam0/camera_info"
  ros_type_name: "sensor_msgs/msg/CameraInfo"
  gz_type_name: "gz.msgs.CameraInfo"

# Right camera
- topic_name: "/model/x500_stereo_cam/right/image_raw"
  ros_topic_name: "/cam1/image_raw"
  ros_type_name: "sensor_msgs/msg/Image"
  gz_type_name: "gz.msgs.Image"

- topic_name: "/model/x500_stereo_cam/right/camera_info"
  ros_topic_name: "/cam1/camera_info"
  ros_type_name: "sensor_msgs/msg/CameraInfo"
  gz_type_name: "gz.msgs.CameraInfo"

# IMU
- topic_name: "/model/x500_stereo_cam/stereo_imu"
  ros_topic_name: "/imu0"
  ros_type_name: "sensor_msgs/msg/Imu"
  gz_type_name: "gz.msgs.IMU"
```

### 3. Launch Bridge

```bash
ros2 run ros_gz_bridge parameter_bridge --ros-args \
  -p config_file:=/path/to/stereo_vins_bridge.yaml
```

## VINS-Fusion Configuration

### Camera Calibration File

Create `stereo_camera_calib.yaml` for VINS-Fusion:

```yaml
%YAML:1.0

#common parameters
imu_topic: "/imu0"
image0_topic: "/cam0/image_raw"
image1_topic: "/cam1/image_raw"
output_path: "~/output/"

# Camera model: PINHOLE or MEI
model_type: PINHOLE
camera_name: stereo_gazebo

# Camera intrinsics
image_width: 752
image_height: 480

# Left camera
distortion_parameters:
   k1: 0.0
   k2: 0.0
   p1: 0.0
   p2: 0.0
projection_parameters:
   fx: 458.654
   fy: 457.296
   cx: 367.215
   cy: 248.375

# Extrinsics: Transformation from camera 0 to camera 1
estimate_extrinsic: 0   # 0: Have accurate extrinsic parameters
body_T_cam0: !!opencv-matrix
   rows: 4
   cols: 4
   dt: d
   data: [1.0, 0.0, 0.0, 0.0,
          0.0, 1.0, 0.0, 0.06,
          0.0, 0.0, 1.0, 0.0,
          0.0, 0.0, 0.0, 1.0]

body_T_cam1: !!opencv-matrix
   rows: 4
   cols: 4
   dt: d
   data: [1.0, 0.0, 0.0, 0.0,
          0.0, 1.0, 0.0, -0.06,
          0.0, 0.0, 1.0, 0.0,
          0.0, 0.0, 0.0, 1.0]

# IMU parameters (adjust based on your IMU)
acc_n: 0.017          # accelerometer measurement noise standard deviation
gyr_n: 0.009          # gyroscope measurement noise standard deviation
acc_w: 0.001          # accelerometer bias random work noise standard deviation
gyr_w: 0.00009        # gyroscope bias random work noise standard deviation
g_norm: 9.81007       # gravity magnitude

#unsynchronization parameters
estimate_td: 0        # online estimate time offset between camera and IMU
td: 0.0               # initial value of time offset

#loop closure parameters
load_previous_pose_graph: 0
pose_graph_save_path: "~/output/pose_graph/"
save_image: 1
```

## Complete Workflow

### Terminal 1: Launch PX4 + Gazebo
```bash
cd ~/workspace/pnk-avoidance/px4
make px4_sitl gz_x500_stereo_cam
```

### Terminal 2: Bridge to ROS2
```bash
source /opt/ros/humble/setup.bash
ros2 run ros_gz_bridge parameter_bridge --ros-args \
  -p config_file:=stereo_vins_bridge.yaml
```

### Terminal 3: Run VINS-Fusion
```bash
source ~/vins_ws/devel/setup.bash  # Or your VINS workspace
roslaunch vins vins_rviz.launch config_path:=/path/to/stereo_camera_calib.yaml
```

### Terminal 4 (Optional): Send VINS Output to PX4
```bash
# Use MAVROS to send VINS odometry back to PX4
rosrun mavros_extras vision_pose_estimate_publisher
```

## Troubleshooting

### No Images in ROS2
- Check bridge is running: `ros2 topic list | grep cam`
- Verify Gazebo topics: `gz topic -l | grep camera`
- Check image transport: `ros2 run image_tools showimage --ros-args -r image:=/cam0/image_raw`

### VINS Not Starting
- Verify camera topics are publishing: `ros2 topic hz /cam0/image_raw`
- Check IMU topic: `ros2 topic hz /imu0`
- Ensure calibration file path is correct

### Poor VINS Performance
- Ensure sufficient lighting in Gazebo world
- Check for textured surfaces (VINS needs visual features)
- Verify synchronization between stereo images
- Tune VINS parameters (feature tracking, optimization)

## Camera Coordinate Frames

- **Gazebo (FLU)**: Forward-Left-Up
- **ROS/VINS (FRD)**: Forward-Right-Down
- **Baseline**: Y-axis (perpendicular to forward direction)

The camera extrinsics in the calibration file account for the 120mm baseline along the Y-axis.

## Model Files

```
x500_stereo_cam/
├── model.sdf           # Gazebo model definition
├── model.config        # Model metadata
└── README.md           # This file
```

## Related Files

- Airframe: `ROMFS/px4fmu_common/init.d-posix/airframes/4030_gz_x500_stereo_cam`

## References

- VINS-Fusion: https://github.com/HKUST-Aerial-Robotics/VINS-Fusion
- Gazebo Documentation: https://gazebosim.org/docs
- PX4 External Vision: https://docs.px4.io/main/en/computer_vision/visual_inertial_odometry.html
