# Hardware-in-the-Loop (HIL) Vehicle Dynamics Co-Simulation: Simulink to CARLA via ROS

## Project Overview
This project demonstrates a closed-loop Hardware-in-the-Loop (HIL) co-simulation environment for autonomous vehicle testing. It bridges the gap between low-level electrical/mechanical physics modeling and high-level 3D environment simulation. 

By bypassing traditional game-engine physics, this architecture allows a highly detailed digital twin of a 300W DC motor and an Electronic Power Steering (EPS) system modeled in **MATLAB/Simulink** to actively drive an ego-vehicle (Tesla Model 3) inside the **CARLA Simulator** via the **Robot Operating System (ROS)**.

## Development Milestones

### 1. The 3:1 Gear Ratio Baseline
The project began by modeling basic rotational mechanics in Simulink. We established a foundational 3:1 mechanical reduction to understand how torque multiplication and angular velocity reduction affect a rotating shaft under load, acting as the baseline for our drivetrain calculations.

### 2. High-Torque 15:1 Gear Ratio
To simulate the physical demands of moving a vehicle's mass from a standstill, the model was upgraded to a 15:1 gear ratio. This milestone focused on calculating the extreme torque required for acceleration and how to map that high-torque, low-speed output to a vehicle's traction limits without inducing tire slip.

### 3. Electrical Drivetrain: 300 Watt DC Motor
We replaced the theoretical torque inputs with a fully modeled electrical drivetrain. We modeled a physical 300W DC Traction Motor powered by a 24V battery system. To solve high-frequency simulation bottlenecks (e.g., simulating a 15 kHz PWM signal running at a 50% duty cycle alongside a 20 FPS game engine), we implemented **Average-Value Modeling**. This calculated the continuous average voltage ($V_{avg} = V_{bat} \times \text{Duty Cycle}$) to run the physics models in real-time without losing mechanical accuracy.

### 4. ROS Integration & Closed-Loop Control
The final milestone bridged the Simulink electrical models with the CARLA game engine. We bypassed CARLA's default Ackermann Python ECU and built a custom closed-loop PID controller directly in Simulink.
* **The Feedback Loop:** Simulink subscribes to the `/carla/ego_vehicle/vehicle_status` ROS topic to read the live vehicle velocity in $m/s$.
* **The Math:** The digital twin calculates the target speed, compares it to the live telemetry, and calculates the error.
* **The Actuation:** The PID controller splits the effort into normalized $(0.0 - 1.0)$ `throttle` and `brake` signals, broadcasting them alongside EPS steering data to the `/carla/ego_vehicle/vehicle_control_cmd` topic to physically drive the 3D car.

## Key Parameters & Variables

### Physical Motor & Electrical Parameters
* **Battery Source ($V_{bat}$):** 24V DC
* **Motor Power Rating:** 300 Watts
* **PWM Switching Frequency:** 15 kHz (Bypassed via Average-Value Modeling for real-time execution)
* **Duty Cycle ($D$):** 0.5 (yielding a 12V effective continuous source)
* **Tire Radius ($r$):** 0.13 Meters (used to convert angular velocity $\omega$ to linear velocity $v$)

### ROS Bridge Formatting Variables
* **Message Type:** `carla_msgs/CarlaEgoVehicleControl` (Replaced standard `ackermann_msgs` for raw pedal access)
* **Data Types:** Strict enforcement of 32-bit `single` floats for ROS compatibility to prevent Simulink `double` propagation crashes.
* **Steer Clamping:** $\pm 0.7$ radians (approx. $\pm 40^\circ$) physical steering lock, mathematically scaled by a factor of `1/0.7` to normalize the output to the $\pm 1.0$ limit required by CARLA.

### Closed-Loop PID Tuning
* **Proportional ($P$):** 1.0 (Scaled up dynamically based on vehicle mass and resting friction)
* **Integral ($I$):** 0.1
* **Derivative ($D$):** 0.0
* **Throttle Output:** Saturation clamped at `[0.0, 1.0]`
* **Brake Output:** Gain inverted `[-1]`, saturation clamped at `[0.0, 1.0]`

## Tech Stack
* **MATLAB / Simulink:** Simscape Electrical, ROS Toolbox
* **ROS 1 (Noetic):** `carla_ros_bridge`
* **Simulation Engine:** CARLA (Unreal Engine 4)
* **OS:** Ubuntu 20.04 (WSL2)

---
