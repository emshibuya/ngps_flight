# Non-GPS Navigation Documentation

Setup and configuration guide for GPS-free navigation using visual-inertial systems with ArduPilot.

**Original Documentation:** [Google Docs](https://docs.google.com/document/d/1Opsji8ZT2YeRjR8lPMAb49Ai44oMDmwoFYnSJPwfXz0/edit?tab=t.0#heading=h.d9zawrpqff1s)

## Hardware Requirements

- Flight controller running ArduPilot (Pixhawk, Cube, etc.)
- Down-facing camera (mono or stereo)
- IMU sensor (integrated in flight controller or external)
- Companion computer (for running ROS2 nodes)
- Calibrated camera-IMU system (see CAMERA_IMU_CALIBRATION.md)

## Software Requirements

- ArduPilot firmware (Copter, Plane, or Rover)
- ROS2 (Humble or later)
- MAVROS2 or MAVLink
- NGPS, UKF, and VIO packages from this repository

## Wiring & Connection

### Camera Connection

- Connect camera to companion computer via USB, Ethernet, or MIPI CSI
- Ensure camera provides timestamped images
- Configure camera frame rate (typically 10-30 Hz)

### IMU Connection

- Use flight controller's onboard IMU (recommended)
- Or connect external IMU via I2C/SPI to companion computer
- Ensure IMU data is accessible via ROS2 topics

### Companion Computer to Flight Controller

- Connect via serial (USB, UART) or Ethernet
- Configure MAVLink connection
- Set appropriate baud rate (921600 or higher recommended)

## ArduPilot Configuration

EKF3 must be the active estimator: `AHRS_EKF_TYPE = 3` and `EK3_ENABLE = 1` (the Copter defaults).

### External position input (`VISO_*`)

ArduPilot receives NGPS through its visual odometry interface, so these are `VISO_*` parameters,
although NGPS provides an absolute position rather than odometry.

| Parameter | Value | Notes |
|---|---|---|
| `VISO_TYPE` | **1** (MAVLink) | Do not use 2 (IntelT265) |
| `VISO_DELAY_MS` | measured latency | Time from image capture to `/ap/tf` arrival, 250 ms at most |
| `VISO_QUAL_MIN` | 0 | |

### EKF source sets

Keep GPS as the primary source set and put NGPS in the secondary set.

| Parameter | Set 1: GPS | Set 2: NGPS |
|---|---|---|
| `EK3_SRCn_POSXY` | 3 (GPS) | **6 (ExternalNav)** |
| `EK3_SRCn_VELXY` | 3 (GPS) | 0 (None) |
| `EK3_SRCn_POSZ` | 1 (Baro) | 1 (Baro) |
| `EK3_SRCn_VELZ` | 3 (GPS) | 0 (None) |
| `EK3_SRCn_YAW` | 1 (Compass) | 1 (Compass) |

`n` is 1 for set 1 and 2 for set 2. NGPS sends position only, so set 2 takes no velocity from it.
Keep yaw on the compass.

### Switching source sets from the transmitter

Assign an RC channel to the EKF source set switch:

```
RC7_OPTION = 90    # EKF Source Set (any free channel works)
```

| Switch position | Source set |
|---|---|
| Low | 1 (GPS) |
| Middle | 2 (NGPS) |
| High | 3 (unused here) |

In SITL, flip the switch from MAVProxy:

```
rc 7 1500    # middle: set 2, NGPS
rc 7 1000    # low: set 1, GPS
```

To compare NGPS against GPS without flying on it, leave the switch low. ArduPilot still logs every
`/ap/tf` sample as `VISP`, and the `GPS` message keeps the true position.

## MAVROS Configuration

### Basic Setup

1. Install MAVROS2:
   ```bash
   sudo apt install ros-humble-mavros ros-humble-mavros-extras
   ```

2. Configure connection:
   ```bash
   ros2 run mavros mavros_node \
     --ros-args \
     -p fcu_url:=/dev/ttyUSB0:921600 \
     -p system_id:=1 \
     -p component_id:=1
   ```

### Topic Mapping

MAVROS publishes/subscribes to:
- `/mavros/odometry/out`: Fused odometry output (to ArduPilot)
- `/mavros/imu/data`: IMU data (from ArduPilot, optional)
- `/mavros/vision_pose/pose`: Vision position estimate (to ArduPilot)

## Enabling and Configuring IMU & Camera Data Streams

### Methods to Enable High-Resolution IMU

#### Method 1: Via MAVLink Streams

In ArduPilot, configure high-rate IMU streams:
```
SR0_ADSB = 0
SR0_EXT_STAT = 2
SR0_EXTRA1 = 10
SR0_EXTRA2 = 10
SR0_EXTRA3 = 2
SR0_PARAMS = 10
SR0_POSITION = 3
SR0_RAW_SENS = 2
SR0_RC_CHAN = 0
```

#### Method 2: Via Companion Computer

Read IMU directly from flight controller via I2C/SPI if supported.

### Full Setup Steps

1. **Start MAVROS:**
   ```bash
   ros2 launch mavros apm.launch.py fcu_url:=/dev/ttyUSB0:921600
   ```

2. **Start your VIO system** (external package; must publish relative pose/odometry).

3. **Start NGPS localization:**
   ```bash
   ros2 launch ap_ngps_ros2 ngps_localization.launch.py \
     reference_image_path:=/path/to/satellite_image.tif
   ```

4. **Start UKF fusion:**
   ```bash
   ros2 launch ap_ukf estimator.launch.py
   ```

5. **Verify topics are publishing:**
   ```bash
   ros2 topic list
   ros2 topic echo /ukf/odometry
   ```

## Calibration Procedure

### Steps

1. **Calibrate Camera-IMU system** (see CAMERA_IMU_CALIBRATION.md)
   - Camera intrinsics
   - IMU noise parameters
   - Camera-IMU extrinsics
   - Time synchronization

2. **Configure ArduPilot parameters:**
   - Set EKF3 as primary estimator
   - Configure external navigation sources
   - Set appropriate noise parameters

3. **Test on ground:**
   - Power on system
   - Verify all ROS2 nodes are running
   - Check that odometry is being published
   - Verify ArduPilot is receiving external navigation

4. **Initial flight test:**
   - Start in manual/stabilize mode
   - Verify position estimates are reasonable
   - Check for drift and accuracy
   - Monitor EKF health in Mission Planner/QGroundControl

5. **Tune parameters:**
   - Adjust EKF noise parameters if needed
   - Tune UKF fusion weights
   - Optimize NGPS matching parameters

## Notes

- Always test in a safe, open area first
- Keep GPS enabled initially for comparison and fallback
- Monitor EKF health and innovation values
- External navigation requires good camera visibility and lighting
- System performance depends on altitude (higher is better for NGPS)
- Ensure adequate computational resources on companion computer
- Regular calibration recommended (every few months or after hardware changes)

## Troubleshooting

### ArduPilot Not Receiving Position

- Check MAVROS connection: `ros2 topic echo /mavros/state`
- Verify odometry topic is publishing: `ros2 topic echo /ukf/odometry`
- Check EKF3 source parameters in ArduPilot

### Poor Position Accuracy

- Verify camera-IMU calibration
- Check camera focus and image quality
- Ensure adequate lighting conditions
- Verify reference image quality and georeferencing

### High Drift

- Check IMU noise parameters
- Verify time synchronization
- Ensure sufficient visual features in camera view
- Check for camera vibration or motion blur

### System Crashes

- Monitor companion computer CPU/memory usage
- Reduce camera frame rate if needed
- Optimize feature matching parameters
- Check for overheating

## Additional Resources

- [Original Google Docs Documentation](https://docs.google.com/document/d/1Opsji8ZT2YeRjR8lPMAb49Ai44oMDmwoFYnSJPwfXz0/edit?tab=t.0#heading=h.d9zawrpqff1s)
