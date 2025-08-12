# Turtlesim-on-ROS-2-Jazzy

# ROS 2 Jazzy + turtlesim + Custom Package Setup (Full Steps)

# Prerequisites:
Ubuntu 24.04 (native or WSL2 on Windows)
We will use ROS 2 Jazzy (the official ROS 2 release for Ubuntu 24.04)
We will install turtlesim, run it, explore topics/services, create a custom Python package, and later add automation logic.
----------------------
# 1) Make sure ROS 2 Jazzy is sourced in every shell
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
source ~/.bashrc
# Why: This ensures that every time you open a new terminal, ROS 2 commands (ros2) are available without manually sourcing.
----------------------
# 2) Remove any leftover ROS Humble references from your bashrc
grep -n "ros/humble" ~/.bashrc
sed -i '/ros\/humble\/setup.bash/d' ~/.bashrc
source ~/.bashrc
# Why: ROS Humble is NOT compatible with Ubuntu 24.04, and having its sourcing line will cause "No such file" errors.
----------------------
# 3) Install ROS 2 Jazzy Desktop
sudo apt update
sudo apt install -y ros-jazzy-desktop
# Why: The "desktop" variant includes core ROS 2 packages, visualization tools (RViz), demos, and more — a full dev environment.
----------------------
# 4) Install turtlesim package
sudo apt install -y ros-jazzy-turtlesim
# Why: turtlesim is a beginner-friendly simulation package for learning ROS 2 concepts like nodes, topics, and services.
----------------------
# 5) Install colcon build system
sudo apt install -y python3-colcon-common-extensions
# Or alternative (depending on apt availability):
sudo apt install -y colcon
# Why: colcon is the build tool used for compiling ROS 2 workspaces and packages.
----------------------
# 6) Create a new ROS 2 workspace
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
colcon build
echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
source ~/.bashrc
# Why: The workspace (ros2_ws) will hold all our custom packages. Sourcing the install/setup.bash ensures local packages are visible.
----------------------
# 7) Run turtlesim (first terminal)
ros2 run turtlesim turtlesim_node
# Why: Launches the turtlesim simulation window with the default turtle1, ready to receive velocity commands.
----------------------
# 8) Control the turtle via keyboard (second terminal)
ros2 run turtlesim turtle_teleop_key
# Why: Publishes geometry_msgs/Twist messages to /turtle1/cmd_vel to move the turtle interactively with arrow keys.
----------------------
# 9) Explore ROS 2 topics and services (optional but highly recommended)
ros2 topic list
ros2 topic echo /turtle1/pose
ros2 service list
ros2 service call /clear std_srvs/srv/Empty "{}"
ros2 service call /turtle1/set_pen turtlesim/srv/SetPen "{r: 255, g: 0, b: 0, width: 3, off: 0}"
ros2 service call /spawn turtlesim/srv/Spawn "{x: 5.5, y: 5.5, theta: 0.0, name: 'turtle2'}"
ros2 service call /kill turtlesim/srv/Kill "{name: 'turtle2'}"
# Why: Understanding how to list, inspect, and call topics/services is fundamental to ROS 2 development.
----------------------
# 10) Create a new Python ROS 2 package (without adding code yet)
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python turtle_controller --dependencies rclpy geometry_msgs turtlesim std_srvs
mkdir -p turtle_controller/turtle_controller
# Why: This package will contain our Python node for automated turtle control (e.g., drawing a square, circle, etc.).
----------------------
# 11) Prepare setup.py for an executable entry point
 Open the file turtle_controller/setup.py and add:
 entry_points={
     'console_scripts': [
         'draw_square = turtle_controller.draw_square:main',
     ],
},
# Why: This defines a ROS 2 command (ros2 run turtle_controller draw_square) that will execute our Python node later.
----------------------
# 12) Build the workspace after adding/modifying packages
cd ~/ros2_ws
rm -rf build install log
colcon build --symlink-install
source ~/ros2_ws/install/setup.bash
# Why: Any new package or code change requires rebuilding the workspace and re-sourcing the environment.
----------------------
# 13) Correct execution order when running
# Terminal 1:
ros2 run turtlesim turtlesim_node
# Terminal 2 (after we add code later):
ros2 run turtle_controller draw_square
# Why: turtlesim_node must be running so our custom node can send velocity commands to it.
----------------------
# 14) GitHub upload (once package is functional)
cd ~/ros2_ws
echo "build/" > .gitignore
echo "install/" >> .gitignore
echo "log/" >> .gitignore
git init
git add src/turtle_controller .gitignore
git commit -m "ROS 2 Jazzy + turtlesim setup + custom package scaffolding"
git branch -M main
git remote add origin https://github.com/<USERNAME>/smartmethods-wk5-day3-turtlesim-ros2.git
git push -u origin main
# Why: Version control and documentation on GitHub is essential for tracking progress and delivering the task.
----------------------
# 15) Common issues encountered and how we fixed them:

Issue: "Unable to locate package ros-humble-desktop" on Ubuntu 24.04
Fix: Installed ros-jazzy-desktop instead.
Issue: "-bash: /opt/ros/humble/setup.bash: No such file"
Fix: Removed Humble source line from ~/.bashrc and replaced with Jazzy.
Issue: "ros2" command not found in PowerShell
Fix: Open Ubuntu/WSL terminal or run "wsl -d Ubuntu".
Issue: Two turtlesim windows spawning
Fix: Only run turtlesim_node in ONE terminal.
Issue: "No executable found" when running custom package
Fix: Ensure setup.py entry_points is correct, rebuild, and source the workspace.
----------------------
# 16) Installed packages summary:
ros-jazzy-desktop
ros-jazzy-turtlesim
python3-colcon-common-extensions (or colcon)
Workspace environment sourced via /opt/ros/jazzy/setup.bash and ~/ros2_ws/install/setup.bash
----------------------------------------------------------------------
## Code I use it for the task

turtle_controller/turtle_controller/draw_square.py
#!/usr/bin/env python3
import math, time, rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from std_srvs.srv import Empty
from turtlesim.srv import SetPen

class DrawSquare(Node):
    def __init__(self):
        super().__init__('draw_square')
        self.pub = self.create_publisher(Twist, '/turtle1/cmd_vel', 10)
        self.clear = self.create_client(Empty, '/clear')
        self.pen = self.create_client(SetPen, '/turtle1/set_pen')
        while not self.clear.wait_for_service(timeout_sec=1.0): pass
        while not self.pen.wait_for_service(timeout_sec=1.0): pass
        p = SetPen.Request(); p.r=0; p.g=0; p.b=255; p.width=4; p.off=0
        self.pen.call_async(p); self.clear.call_async(Empty.Request())
        self.draw(2.0, 2.0, 2.0); rclpy.shutdown()

    def move(self, secs, v):
        m = Twist(); m.linear.x = v; t = time.time()+secs
        while time.time()<t and rclpy.ok(): self.pub.publish(m); time.sleep(0.01)
        self.pub.publish(Twist())

    def turn(self, rad, w):
        m = Twist(); m.angular.z = w if rad>0 else -w; t = time.time()+abs(rad)/w
        while time.time()<t and rclpy.ok(): self.pub.publish(m); time.sleep(0.01)
        self.pub.publish(Twist())

    def draw(self, side, lin, ang):
        for _ in range(4): self.move(side, lin); self.turn(math.pi/2, ang)

def main(): rclpy.init(); DrawSquare()
if __name__=='__main__': main()
---------------------------------------------------------------------
turtle_controller/setup.py (only the entry_points part edited)

entry_points={
    'console_scripts': [
        'draw_square = turtle_controller.draw_square:main',
    ],
},
---------------------------------------------------------------------

## Terminal 1 – start turtlesim node
source ~/ros2_ws/install/setup.bash
ros2 run turtlesim turtlesim_node

-------------------------------------
## Terminal 2 – run the custom draw_square node

source ~/ros2_ws/install/setup.bash
ros2 run turtle_controller draw_square
