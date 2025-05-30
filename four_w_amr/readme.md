# VEEROBOT Motor Control Firmware

## Overview

This firmware provides robust motor control for the VEEROBOT platform, an ESP32-based robot with four motors and quadrature encoders. It features dual control modes, configurable ramping, and a ROS-compatible command interface.

![VEEROBOT Platform](https://via.placeholder.com/800x400?text=VEEROBOT+Platform)

## Key Features

- **Dual Control Modes**:
  - **Direct Mode**: Simple PWM control with predictable response
  - **PID Mode**: Precise velocity control using encoder feedback

- **Motion Control**:
  - Configurable ramping for smooth acceleration/deceleration
  - Motor stall protection with minimum effective speed
  - Immediate or ramped stopping options

- **Safety Features**:
  - Watchdog timeout (configurable, default 15s)
  - Emergency stop commands
  - Multiple layers of motor stopping on reset/startup

- **User Interface**:
  - Case-insensitive command parsing
  - Detailed help system
  - Configurable logging levels

- **ROS Integration**:
  - Compatible with ROS differential drive controller
  - Encoder feedback for odometry
  - Status feedback for debugging

## Hardware Requirements

- ESP32 microcontroller
- PCA9685 PWM driver
- 4 DC motors with quadrature encoders
- Motor drivers (H-bridges)

## Pin Configuration

Pin configuration is managed in `lib/lyraConfig/src/lyraConfig.cpp`:

```cpp
// Motor configuration: {IN1, IN2, PWM, REVERSED}
const MotorConfig MOTOR_CONFIG[4] = {
    {10, 9, 8, true},    // Front Left Motor
    {11, 12, 13, true},  // Front Right Motor
    {4, 3, 2, false},    // Rear Left Motor
    {5, 6, 7, false}     // Rear Right Motor
};

// Encoder configuration: {PIN_A, PIN_B, REVERSED}
const EncoderConfig ENCODER_CONFIG[4] = {
    {14, 27, true},  // Front Left Encoder
    {25, 26, true},  // Front Right Encoder
    {17, 13, false}, // Rear Left Encoder
    {18, 19, false}  // Rear Right Encoder
};
```

## Serial Command Interface

### Control Commands

| Command | Description |
|---------|-------------|
| `CMD <left> <right>` | Set motor speeds (-1.0 to 1.0) |
| `CMD 0 0` | Stop motors (immediate or ramped based on setting) |
| `STOP` or `E-STOP` | Emergency stop (always immediate) |
| `FULL` or `FULL SPEED` | Apply maximum forward speed immediately |
| `MOTOR TEST` | Run motor test sequence for diagnostics |

### Configuration Commands

| Command | Description |
|---------|-------------|
| `MODE DIRECT` | Use direct motor control (PWM) |
| `MODE PID` | Use PID velocity control |
| `RAMP ON` | Enable speed ramping |
| `RAMP OFF` | Disable speed ramping |
| `RAMP RATE <value>` | Set ramp speed (0-1.0)<br>Lower values (0.01-0.05) = smoother acceleration<br>Higher values (0.1-1.0) = faster acceleration |
| `IMMEDIATE STOP ON` | `CMD 0 0` stops motors instantly (default) |
| `IMMEDIATE STOP OFF` | `CMD 0 0` ramps down to stop using current ramp rate |
| `LOG NONE/ERROR/INFO/DEBUG` | Set logging level |
| `HELP` | Show available commands |

### Status Messages

When log level is set to INFO or DEBUG, the firmware outputs:

```
ENC <fl> <fr> <rl> <rr>      # Encoder tick values
VEL <fl> <fr> <rl> <rr>      # Wheel velocities (mm/s)
MODE <DIRECT/PID>[+RAMP]     # Current control mode
```

## ROS Integration

### ROS Setup

1. Install the `rosserial_arduino` package:
   ```bash
   sudo apt-get install ros-$ROS_DISTRO-rosserial-arduino
   sudo apt-get install ros-$ROS_DISTRO-rosserial
   ```

2. Install the PlatformIO extension for VS Code

3. Clone this repository:
   ```bash
   git clone https://github.com/username/veerobot-motor-control.git
   ```

4. Open the project in VS Code with PlatformIO

### ROS Node Example

Below is an example ROS node to control the robot:

```python
#!/usr/bin/env python
import rospy
import serial
from geometry_msgs.msg import Twist
from nav_msgs.msg import Odometry
from sensor_msgs.msg import JointState

class VeerobotController:
    def __init__(self):
        # Initialize ROS node
        rospy.init_node('veerobot_controller')
        
        # Parameters
        self.port = rospy.get_param('~port', '/dev/ttyUSB0')
        self.baud = rospy.get_param('~baud', 115200)
        self.max_linear = rospy.get_param('~max_linear', 0.5)  # m/s
        self.max_angular = rospy.get_param('~max_angular', 1.0)  # rad/s
        
        # Open serial connection
        self.ser = serial.Serial(self.port, self.baud, timeout=1)
        rospy.sleep(2)  # Wait for Arduino to reset
        
        # Initialize robot
        self.ser.write(b"MODE DIRECT\n")
        self.ser.write(b"RAMP ON\n")
        self.ser.write(b"RAMP RATE 0.05\n")
        self.ser.write(b"LOG INFO\n")
        
        # Publishers
        self.odom_pub = rospy.Publisher('odom', Odometry, queue_size=10)
        self.joint_pub = rospy.Publisher('joint_states', JointState, queue_size=10)
        
        # Subscribers
        rospy.Subscriber('cmd_vel', Twist, self.cmd_vel_callback)
        
        # Timer for reading from serial
        rospy.Timer(rospy.Duration(0.1), self.read_serial)
        
        rospy.loginfo("VEEROBOT controller initialized")
    
    def cmd_vel_callback(self, msg):
        # Convert Twist message to differential drive commands
        linear = max(min(msg.linear.x, self.max_linear), -self.max_linear)
        angular = max(min(msg.angular.z, self.max_angular), -self.max_angular)
        
        # Calculate left and right wheel velocities
        left = linear - angular * 0.5  # Adjust based on robot width
        right = linear + angular * 0.5
        
        # Normalize to -1.0 to 1.0 range
        left = left / self.max_linear
        right = right / self.max_linear
        
        # Clamp values
        left = max(min(left, 1.0), -1.0)
        right = max(min(right, 1.0), -1.0)
        
        # Send command to robot
        cmd = f"CMD {left:.2f} {right:.2f}\n"
        self.ser.write(cmd.encode())
    
    def read_serial(self, event):
        if self.ser.in_waiting:
            line = self.ser.readline().decode('utf-8').strip()
            if line.startswith("ENC"):
                parts = line.split()
                if len(parts) == 5:  # "ENC fl fr rl rr"
                    # Process encoder values for odometry
                    pass
            elif line.startswith("VEL"):
                parts = line.split()
                if len(parts) == 5:  # "VEL fl fr rl rr"
                    # Process velocity values
                    pass
    
    def shutdown(self):
        # Stop the robot
        self.ser.write(b"CMD 0 0\n")
        self.ser.close()

if __name__ == '__main__':
    controller = VeerobotController()
    rospy.on_shutdown(controller.shutdown)
    rospy.spin()
```

### ROS Launch File Example

```xml
<launch>
  <!-- Serial connection to VEEROBOT -->
  <node name="veerobot_controller" pkg="veerobot_controller" type="controller.py" output="screen">
    <param name="port" value="/dev/ttyUSB0" />
    <param name="baud" value="115200" />
    <param name="max_linear" value="0.5" />
    <param name="max_angular" value="1.0" />
  </node>
  
  <!-- Robot state publisher -->
  <param name="robot_description" textfile="$(find veerobot_description)/urdf/veerobot.urdf" />
  <node name="robot_state_publisher" pkg="robot_state_publisher" type="robot_state_publisher" />
  
  <!-- Teleop keyboard for testing -->
  <node name="teleop_twist_keyboard" pkg="teleop_twist_keyboard" type="teleop_twist_keyboard.py" output="screen" />
</launch>
```

## Motor Testing Procedure

1. Connect to the ESP32 using a serial terminal at 115200 baud
2. Type `MOTOR TEST` to run the diagnostic sequence
3. For individual motor testing:
   - Use `MODE DIRECT` to enable direct control
   - Use `CMD 0.3 0` to test left motors at 30% power
   - Use `CMD 0 0.3` to test right motors at 30% power
   - Use `CMD 0 0` to stop

## Tuning Recommendations

### PID Tuning

1. Start with low values: Kp = 0.1, Ki = 0, Kd = 0
2. Modify values in `lib/lyraConfig/src/lyraConfig.h`
3. Increase Kp until oscillation occurs, then reduce slightly
4. Add Ki to eliminate steady-state error
5. Add Kd to reduce overshoot if necessary

### Ramping Tuning

For smooth acceleration/deceleration:
- Use `RAMP RATE 0.01` to `RAMP RATE 0.05` (smoother, slower)
- Use `RAMP RATE 0.1` to `RAMP RATE 0.3` (faster, less smooth)
- Use `RAMP RATE 0` or `RAMP OFF` for immediate speed changes

## FreeRTOS Considerations

The current implementation uses a simple loop-based approach. For more complex applications, especially when integrating with IMUs or other sensors, a FreeRTOS implementation would provide:

- **Task prioritization**: Ensure critical control loops run on time
- **Better resource management**: When dealing with multiple sensors and communication channels
- **Simplified timing**: Using task notification and synchronization primitives

To implement FreeRTOS with this firmware:
1. Create separate tasks for:
   - Command processing
   - Motor control
   - Encoder readings
   - ROS communication
   - IMU or additional sensor processing
2. Use queues for inter-task communication
3. Use semaphores for resource protection

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Credits

Developed for VEEROBOT by [Your Name/Organization]

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request. 
