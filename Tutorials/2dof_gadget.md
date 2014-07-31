---
layout: page
title: 2-DOF Gadget
---
An IMU Driven 2-DOF Gadget using Microblx
====

##Setup

The setup consists of three devices:

1. A ZedBoard, Zynq Series FPGA SoC with Linux OS
2. An IMU 
3. A 2-DOF gadget


In the setup, the IMU and the 2-DOF gadget are connected to the ZedBoard through expansion headers. Dedicated drivers built with FPGA resources are in charge of reading mesured data from The IMU and driving the servo motors on the gadget. Both drivers are wrapped as function blocks (IMU Block and PWM Block in the figure below) in Microblx software framework, and are accessible by memory mapping. 

![Pins](ubx_fpga_2dof.png )
	

###IMU 

The accelerometer on the IMU is used for orientation detection. The IMU block returns the angles between x-y, x-z and y-z axes in a 3 element array of float type, ranging from 0 to 360 degrees. 

###Servo motor
The HS485-HB servo motors are driven by 20Hz PWM signals. The position of the motor is controlled by PWM pulse width. A pulse width of 1442 us keeps the motor stays at 0 degree; 601us and 2283us are corresponding to -90 and + 90 degrees. The PWM generators use 100MHz clock, each clock cycle is 10ns. Therefore, the internal counter needs to count to 100 for 1us. To keep the motor at 0 degree, 144200 (representing 1442us) should be sent to the PWM Block.

##Demo
The objective of is to control the 2-DOF gadget using the IMU as a joystick. Lua script *examples/imu_balance.lua* runs a complete demo. 
[video link](http://people.mech.kuleuven.be/~lin.zhang/_Media/ubx/imu_control.mp4) 

The IMU and motor drivers are provided as function blocks in directory **std_block**: *imu_driver* and *servo_motor_driver*. Source code for *imu_balance* can be found in std_block, too. 

User could modify code in *std_block/gadget* (a copy of imu_balance) and create a lua script *examples/gadget.lua* to make the gadget controller. The following linear relationship should be considered:

	(180, 360) degrees on IMU <--> (60100, 228300) clock cycles for PWM generation <--> (-90, 90) degrees of servo motor
 
##Access to the ZedBoard
The IP addresses of the ZedBoards are:
	
	192.168.10.66
	192.168.10.67
	192.168.10.68
	192.168.10.69

Use **SSH** to login to the FPGA, both username and password are “linaro”. 

	$ ssh linaro@192.168.10.66
	
Go to the microblx directory:
	
	$ cd /home/linaro/robotics/ubx_tutorial

Source the environment file: 
	
	$ . env.sh
	
Build the blocks:

	$ make
	
To run the demo:
	
	$ luajit -i examples/imu_balance.lua

The web interface is available for details:

	http://192.168.10.66:8888
