# Control allocation using Machine Learning

This repository implements the control allocation (mixing) with machine learning in PX4. A ML model is trained in Python (google colabs) using Keras, deployed in the PX4 code using the fdeep library and tested using the mrs sitl environment. Although the implementation is different, the methodology was inspired by [this paper](https://ieeexplore.ieee.org/document/9536649).

This rep can be used as reference to implement and deploy ML models in PX4 firmware.

## Overview

The basic idea is to create a machine learning pipeline that can be optimize to increase motor mixing performance. 

The legacy PX4 motor mixing uses a set of geometric information from the multicopter frame to calculate the appropriate thrust of each actuator (motor + propeller) that applies a certain desired force/torque in the multicopter frame, given by the control pipeline. Then, this actuator thrust is converted to a Normalized Actuator Command (NAC) in the interval [-1, 1] using a simple model. This command is then sent to the output_drivers (PWM, UAVCAN, etc..).

A simple MLP model that mimics the behavior of the legacy PX4 mixing was implemented as proof of concept.

## Instalation

The project is organized as a set of repositories that were forked from the [ctu-mrs project](https://ctu-mrs.github.io/). The reps forked are: 

- [simulation](https://github.com/davihdsantos/simulation)
- [mrs_simulation](https://github.com/davihdsantos/mrs_simulation)
- [px4_firmware](https://github.com/davihdsantos/px4_firmware/tree/control_allocation_nn)
    - [NuttX](https://github.com/davihdsantos/NuttX/tree/control_allocator_laser)

The main rep is the px4_firmware, where the majority of the changes are made.

The easiest way to install is:

1) install the mrs system as indicated in their [main rep](https://github.com/ctu-mrs/mrs_uav_system);
2) install the forked reps:

    ```console
    cd ~/mrs_workspace/src/simulation
    git remote set-url origin https://github.com/davihdsantos/simulation
    git pull
    gitman update
    ```
    
    
    this should install the forked submodules described in gitman.yml.

    If this instalation process dosent works, you can set every remote repository individually. Example:

    ```console
    cd ~/mrs_workspace/src/simulation/ros_packages/px4_firmware
    git remote set-url origin https://github.com/davihdsantos/px4_firmware
    git pull
    git checkout control_allocation_nn
    ```
    > **Note** <br>
    > NuttX git locaton is *px4_firmware/platforms/nuttx/NuttX/nuttx* and the branch is **control_allocation_laser**
    
    > **Note** <br>
    > px4_firmware branch is **control_allocation_nn** and NuttX branch is **control_allocation_laser**

3) install the [frugally deep library](https://github.com/Dobiasd/frugally-deep#requirements-and-installation). this library allows deployment of .h5 models in c++.

4) follow [these instructions](https://github.com/davihdsantos/px4_firmware/blob/control_allocation_nn/src/examples/nn_example/README.md) to generate a json description of the .h5 that is used by the fdeep library.

    > **Warning** <br>
    > the .json must be placed at *~/.ros/sitl_uav1/* since this is the base directory for the running sitl PX4 firmware, otherwise you will get the error "cannot open fdeep_model.json".

5) Then compile the mrs_workspace:

    ```console
      cd ~/mrs_workspace
      catkin build
    ```

### Running

```console
  cd ~/mrs_workspace/src/simulation/example_tmux_scripts/get_nn_data
  ./start.sh
```

## How it works (end-to-end)

### Getting the data

The first step is to get the data to train the ML model. This can be done in several ways: you can use a joystick to move the drone around (see [rc_teleop](https://github.com/LASER-Robotics/rc_teleop)), or you can use a PX4 module to publish commands directly to **actuator_commands** topic, as it is done [here](https://github.com/davihdsantos/px4_firmware/blob/1.11.2/src/examples/send_cmd_to_motor/send_cmd_to_motor.c). To run this module just type *send_cmd_to_motor* in the console (**gazebo** window in tmux).

When you get enough data, use *commander land* in the px4 console to land the drone, disarm and save the log file. The log is saved at *~/.ros/sitl_uav1/log/*. 

### Getting the .h5 model

Use the command *ulog2csv* to get the csv files from the ulog. 

Then, use two colabs: prepare_data and train_nn. prepare_data is used to prepare the data for training and train_nn is used to train and evaluate the NN model. This generates a h5 file in your driver.

> **Note** <br>
> you can find a detailed explanation [here](https://github.com/davihdsantos/px4_firmware/blob/control_allocation_nn/src/examples/nn_example/README.md).

> **Note** <br>
> the prepare_data.ipynb expects a file named *.csv. this is not the name of the file after running ulog2csv, so you might have to rename the files (remove the _0 at the end) and/or open it with google sheets in your driver (and rename it).

> **Warning** <br>
> pay attention to the path that the colabs expect to find the files. the files for prepare_data.ipynb (csv files) are in */content/drive/My\ Drive/UFPB/control_allocation_nn/data_train/first_experiment/raw/* in my driver, but you can change that to a more convinient location. The same with the tran_nn.ipynb colab.

### Deploying the model

Download the h5 and convert it to a json, as instructed [here](https://github.com/davihdsantos/px4_firmware/blob/control_allocation_nn/src/examples/nn_example/README.md). The .json must be placed at *~/.ros/sitl_uav1/*

The main code is located at *px4_firmware/src/lib/mixer/MultirotorMixer/MultirotorMixer.cpp* line 338. The legacy mixer is executed by updateValues() and the ML mixer is executed by updateValuesNN(). Just uncomment the one you want to use.

The function *printOutputs(rate, outputs)* prints the NAC in the console.
