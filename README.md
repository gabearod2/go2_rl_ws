# RL Control and Navigation Deployment - Unitree Go2

## Introduction and Scope

This repository contains a ROS2 Humble workspace allowing for the real world implementation of a trained RL policy on the Go2. Currently, there is support for locomotion policies and 2D navigation policies (without obstacle avoidance). To train the RL policies, [Isaac Lab](https://github.com/isaac-sim/IsaacLab) was used.

### Policy Information

Locomotion models can be found in the share directory of the unitree_ros2_python package. Navigation models can be found in the share directory of the rl_navigation package. They are in the ONNX format for maximum compatibility.

Note that the observations tensor has been edited to not include the height scan or the base linear velocities as these are not easily attainable in low state, which the robot has to be in to deliver low level commands. Development is currently underway in LiDAR decoding for height map information and sensor fusion for linear velocity information, as well as support for 3D navigation. 

### FlowCharts

Here is the current configuration of the locomotion policies:

![RL Control FlowChart](https://github.com/gabearod2/go2_rl_ws/blob/main/images/RL%20CONTROL.jpeg)

Here is the current configuration of the navigation policies:

![RL Navigation FlowChart](https://github.com/gabearod2/go2_rl_ws/blob/main/images/RL%20NAVIGATION.jpeg)

## Setup

Before setup, ensure you have installed [ROS2 Humble](https://docs.ros.org/en/humble/Installation.html) and are familiar to connecting your system to the Go2, through ethernet, referring to [Unitree's documentation](https://support.unitree.com/home/en/developer/Quick_start.).

To start, clone this repository into your ROS2 workspaces directory:
```bash
cd ~/workspaces
git clone --recurse-submodules https://github.com/gabearod2/go2_rl_ws
cd go2_rl_ws
conda env create -f environment.yml
conda activate go2_rl_ws_env
```

Resolving dependencies:
```bash
sudo apt install ros-humble-rmw-cyclonedds-cpp
sudo apt install ros-humble-rosidl-generator-dds-idl
```

Ensuring you have not sourced ROS2, compile cyclonedds:
```bash
cd ~/workspaces/go2_rl_ws/src/unitree_ros2/cyclonedds_ws/src
git clone https://github.com/ros2/rmw_cyclonedds -b humble
git clone https://github.com/eclipse-cyclonedds/cyclonedds -b releases/0.10.x
cd cylconedds
git checkout tags/0.10.2 -b my-0.10.2-branch
cd ../..
colcon build --packages-select cyclonedds
```

Source ros and build unitree ROS2:
```bash
source /opt/ros/humble/setup.bash
colcon build
```

Connecting the ethernet cord to the quadruped, get the name of the connection:
```bash
ifconfig
```

After using ifconfig to get name of connection, edit the setup.sh file, using enp114s0 as an example:
```bash
sudo gedit ~/workspaces/go2_rl_ws/src/unitree_ros2/setup.sh
```
```bash
#!/bin/bash
echo "Setup unitree ros2 environment"
source /opt/ros/humble/setup.bash
source ~/workspaces/go2_rl_ws/src/unitree_ros2/cyclonedds_ws/install/setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export CYCLONEDDS_URI='<CycloneDDS><Domain><General><Interfaces>
                            <NetworkInterface name="enp114s0" priority="default" multicast="default" />
                        </Interfaces></General></Domain></CycloneDDS>'
```

Then, compile unitree_go, unitree_api, rl_deploy, go2_launch, rl_navigation, rl_deploy_nav and unitree_ros2_python packages:
```bash
cd ~/workspaces/go2_rl_ws/src/go2_python_sdk2
pip3 install -e .
cd ~/workspaces/go2_rl_ws
source ~/workspaces/go2_rl_ws/src/unitree_ros2/setup.sh
colcon build --packages-select unitree_api &&
colcon build --packages-select unitree_go &&
colcon build --packages-select unitree_ros2_python --symlink-install &&
colcon build --packages-select rl_navigation --symlink-install &&
colcon build --packages-select rl_deploy &&
colcon build --packages-select rl_deploy_nav &&
colcon build --packages-select go2_launch
```

Finally, restart your PC, as recommended by Unitree.

## Body Control Deployment

To deploy, ensure the quadraped is LYING DOWN with SPORT MODE OFF (do so in the app), as support for switching modes is not yet integrated. Tethering the top of the quadruped is also advised. Work is underway for better and safer testing. Here is the wireless remote control mapping:

UP --> Stand Command

DOWN --> Sit Command

START --> Start Walking Command (Deploying RL Actions)

LEFT JOYSTICK --> Linear Velocity Commands

RIGHT JOYSTICK --> Angular Velocity Commands

SELECT --> Stop Walking Command, Goes to Stand Position

A --> Soft Abort, Similir to Damping from Unitree

B --> Kill Command, Only for EMERGENCIES

Open a terminal, source unitree_ros and lse_go2_ws, and launch:
```bash
source ~/workspaces/go2_rl_ws/src/unitree_ros2/setup.sh &&
source ~/workspaces/go2_rl_ws/install/setup.sh &&
cd ~/workspaces/go2_rl_ws &&
ros2 launch go2_launch run_obs_nodes_onnx.launch.py
```

Open a new terminal, and run the low command message publisher:
```bash
source ~/workspaces/go2_rl_ws/src/unitree_ros2/setup.sh &&
source ~/workspaces/go2_rl_ws/install/setup.sh &&
cd ~/workspaces/go2_rl_ws &&
ros2 run rl_deploy go2_rl_control
```

## Navigation Deployment

To deploy, ensure the quadraped is standing in Sport Mode or AI Mode. The pose commands should be sent in the world frame. Ensure that the z command is around 0.35 [m].

Open a terminal, source unitree_ros and lse_go2_ws, and create a pose command:
```bash
cd /mnt/nvme_partition/ros2_ws/go2_rl_ws &&
source /mnt/nvme_partition/ros2_ws/go2_rl_ws/src/unitree_ros2/setup.sh &&
source /mnt/nvme_partition/ros2_ws/go2_rl_ws/install/setup.sh &&
ros2 run rl_navigation go2_pose_command 
```

Open a new terminal, source unitree_ros and lse_go2_ws, and run the projected gravity publisher:
```bash
source ~/workspaces/go2_rl_ws/src/unitree_ros2/setup.sh &&
source ~/workspaces/go2_rl_ws/install/setup.sh &&
cd ~/workspaces/go2_rl_ws &&
ros2 run unitree_ros2_python go2_projected_gravity
```

Open a new terminal, source unitree_ros and lse_go2_ws, and run navigation action inference:
```bash
cd /mnt/nvme_partition/ros2_ws/go2_rl_ws &&
source /mnt/nvme_partition/ros2_ws/go2_rl_ws/src/unitree_ros2/setup.sh &&
source /mnt/nvme_partition/ros2_ws/go2_rl_ws/install/setup.sh &&
ros2 run rl_navigation go2_rl_nav_actions_onnx
```

Open a new terminal, source unitree_ros and lse_go2_ws, and run navigation commands:
```bash
cd /mnt/nvme_partition/ros2_ws/go2_rl_ws &&
source /mnt/nvme_partition/ros2_ws/go2_rl_ws/src/unitree_ros2/setup.sh &&
source /mnt/nvme_partition/ros2_ws/go2_rl_ws/install/setup.sh &&
ros2 run rl_deploy_nav go2_rl_nav
```

## Video of Preliminary Results

The following is a sneak peek into the longer [video](https://youtu.be/o3_ABcsxeG8) of the pronking gait that has currently been achieved:

![Video preview](https://github.com/gabearod2/go2_rl_ws/blob/main/images/pronking.gif)

## Comments and Disclaimer

This repository was developed entirely within Embry-Riddle Aeronautical University's Engineering Physics Propulsion Laboratory! Check us out [here](https://daytonabeach.erau.edu/about/labs/engineering-physics-propulsion-lab). Thank you also to the RoboVerse community!

This is an experimental code, we are not responsible for any damages! Use at your own risk.

## Training

To find the training environment I used through Isaac Lab, follow my forked [Isaac Lab repo](https://github.com/gabearod2/IsaacLab/tree/rl_deployment). To edit which RL policy you use, edit go2_rl_actions.py to use a different ONNX model, ensuring it takes the same input as the current models.  

Future work is to train the quadruped in the the following way: [Extreme Parkour](https://github.com/chengxuxin/extreme-parkour.git) for the best results.
