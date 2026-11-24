# ros2-vicon-jazzy

ROS 2 **Jazzy** port of the OPT4SMART Vicon receiver, forked from
[Noe-Sanchez/ros2-vicon-humble](https://github.com/Noe-Sanchez/ros2-vicon-humble).

Used by `fxteso_ibvs` to aid PX4's EKF2 with mocap. Run it on the **laptop**: the vendored
`DataStreamSDK_10.1/libViconDataStreamSDK_CPP.so` is **x86-64 only** and will not run on the
Raspberry Pi. The Pi runs `vicon_px4_bridge`, which converts this node's `PoseStamped` into
`px4_msgs/VehicleOdometry`.

```sh
colcon build --symlink-install
source install/setup.bash
# LARGE_DATA is NOT optional: every process in fxteso_ibvs's hardware.launch.py runs with it,
# and a LARGE_DATA subscriber receives nothing from a default-transport publisher.
FASTDDS_BUILTIN_TRANSPORTS=LARGE_DATA \
  ros2 run vicon_receiver vicon_client --ros-args \
    -p hostname:=<vicon host> -p port:=801 -p namespace:=vicon
```

Publishes `geometry_msgs/PoseStamped` on `/<namespace>/<subject>/<segment>`, in **metres**, in the
Vicon global frame, stamped with this machine's wall clock. Run **chrony** against the Pi, or the
bridge cannot compute sample age (pass it `stamp_source:=receipt` if you do not).

## Changes from the Humble fork

- **Dropped Boost entirely** (`std::thread`/`std::mutex` instead). The SDK is built against Boost
  1.58 and ships its own stubs; linking the node against system Boost 1.83 as well put 1.83's
  header-only `boost::system` symbols in the executable, where they preempted the SDK's and
  **segfaulted it inside `Connect()`** on Ubuntu 24.04. This was the only change needed to make it
  run, and it is not optional.
- The SDK is found by `find_library` with an RPATH into `DataStreamSDK_10.1/`, so **`install_libs.sh`
  and its `sudo` are no longer needed** - the Humble fork linked the bare library name and silently
  depended on the SDK having been copied into `/usr/lib` first.
- C++17 (Jazzy's `rclcpp` requires it), `cmake_minimum_required` 3.8, and the dead
  `get_default_rmw_implementation()`/`get_rmw_typesupport()` block removed.
- Added a **`port`** parameter; it was previously implicit at 801 and only reachable by writing
  `"host:801"` into `hostname`.
- **Segment data only.** The node enabled marker, unlabeled-marker, marker-ray, device and debug
  streams and read none of them. Here the link carries a flight-control aiding source and must not
  stall: EKF2 drops external vision 400 ms after the last sample it receives.
- On non-x86-64 machines (the Pi) the package configures empty with a warning, so it can sit in a
  shared workspace that is built on both.
- Removed `msg/Position.msg` and the `rosidl` declarations. The package has no
  `rosidl_generate_interfaces()` call and never generated that message - it published
  `PoseStamped` regardless, and the file misled anyone grepping for it.

---

# Vicon receiver for ROS 2 ![GitHub tag (latest by date)](https://img.shields.io/github/v/tag/andreacamisa/ros2-vicon-receiver)

**ros2-vicon-receiver** is a ROS 2 package, written in C++, that retrieves data from Vicon software and publishes it on ROS 2 topics. The code is partly derived from a mixture of [Vicon-ROS2](https://github.com/aheuillet/Vicon-ROS2) and [Vicon bridge](https://github.com/ethz-asl/vicon_bridge).

This is NOT an official ROS 2 package and is not supported. The package has been successfully tested with ROS 2 Dashing Diademata, ROS 2 Foxy, ROS 2 Galactic and ROS 2 Humble on the operating systems Ubuntu 18.04 Bionic Beaver, Ubuntu 20.04 Focal Fossa and MacOS 10.13 High Sierra.

## Requirements

### Installation of dependencies

If you are using Ubuntu 18.04 Bionic Beaver, you can install all the dependencies by simply `cd`'ing into the main project folder and then running
```
$ ./install_ubuntu_bionic.sh
```

Otherwise, proceed as follows. Make sure you have ROS 2 installed (follow the instructions at the [ROS 2 website](https://index.ros.org/doc/ros2/Installation/)).

Then, install [Colcon](https://colcon.readthedocs.io/en/released/index.html) and [CMake](https://cmake.org/) :
```
$ sudo apt install -y python3-colcon-common-extensions cmake
```

### Installation of Datastream SDK and other libraries

The Datastream SDK libraries are required to be installed in the system. You can find them on [the official website](https://www.vicon.com/software/datastream-sdk/?section=downloads).

This package is shipped with Datastream SDK 10.1 (the latest version at the time of writing). If you are running Linux x64 and you want to install this version, simply `cd` into the main project folder and issue the command
```
$ ./install_libs.sh
```

## Quick start

### Building the package

:warning: Do not forget to source the ROS 2 workspace: `source /opt/ros/dashing/setup.bash`

Enter the project folder and build the executable
```
$ cd vicon_receiver
$ colcon build --symlink-install
```

### Running the program

Open a new terminal and source the project workspace:
```
$ source vicon_receiver/install/setup.bash
```

To run the program, use the [launch file template](vicon_receiver/launch/client.launch.py) provided in the package. First, open the file and edit the parameters. Running `colcon build` is not needed because of the `--symlink-install` option previously used.

Now you can the program with
```
$ ros2 launch vicon_receiver client.launch.py
```

Exit the program with CTRL+C.

### Information on ROS 2 topics and messages

The **ros2-vicon-receiver** package creates a topic for each segment in each subject with the pattern `namespace/subject_name/segment_name`. Information is published on the topics as soon as new data is available from the vicon client (typically at the vicon client frequency). The message type [Position](vicon_receiver/msg/Position.msg) is used.

Example: suppose your namespace is the default `vicon` and you have two subjects (`subject_1` and `subject_2`) with two segments each (`segment_1` and `segment_2`). Then **ros2-vicon-receiver** will publish [Position](vicon_receiver/msg/Position.msg) messages on the following topics:
```
vicon/subject_1/segment_1
vicon/subject_1/segment_2
vicon/subject_2/segment_1
vicon/subject_2/segment_2
```

## Constributors
**ros2-vicon-receiver** is developed by
[Andrea Camisa](https://www.unibo.it/sitoweb/a.camisa),
[Andrea Testa](https://www.unibo.it/sitoweb/a.testa) and
[Giuseppe Notarstefano](https://www.unibo.it/sitoweb/giuseppe.notarstefano)

## Acknowledgements
This result is part of a project that has received funding from the European Research Council (ERC) under the European Union's Horizon 2020 research and innovation programme (grant agreement No 638992 - OPT4SMART).

<p style="text-align:center">
  <img src="logo_ERC.png" width="200" />
  <img src="logo_OPT4Smart.png" width="200" /> 
</p>
