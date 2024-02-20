# wolf

## Micro_ROS Setup

Follow this link to install the Micro_ROS agent on the host: [Micro_ROS Installation Guide](https://micro.ros.org/docs/tutorials/core/first_application_linux/)

**_NOTE:_** No need to follow firmware step just need to create agent

## Udev Rules setup

```bash
cd /etc/udev/rules.d
touch 99-my_rules.rules
sudo nano 99-my_rules.rules
```
```bash
ACTION=="add", SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", SYMLINK+="ttyesp32", GROUP="<ADD_GROUP_NAME>", MODE="0660"
```

```bash
sudo chmod 644 99-my_rules.rules 
sudo chown root:root 99-my_rules.rules 
sudo udevadm control --reload-rules        
sudo udevadm trigger
```
## Platformio
Install Platforio on vscode and Upload micro_ros firmware in esp32 (*in micro_ros package*)

## Launch Files

```bash
ros2 launch four_w_amr display.launch.py
ros2 launch four_w_amr bringup.launch.py
```

