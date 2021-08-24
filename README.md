# sciroc2021_objdet_meta
Meta-repository referencing the submodules developed for the Object Detection part of the 2021 SciRoc contest.

Fort further info give a look at the READMEs for the individual repos (**work in progress**).

## Launch

The following launchfile launches all the components in sequence.

```bash
roslaunch sciroc_objdet sciroc_objdet_whole_module.launch
```

## Data flow

To communicate with the module, once it's running, it's necessary to only use its "interface", aka the `objdet_interface` action server. Everything else is carried out by the module itself. Notice the server takes a few seconds to reply, due to its inner workings.
Three different types of requests can be made, depending on the information needed from the module:
- **enumeration** [code 0]: return the number of objects seen
- **classification** [code 1]: return the tags/classes of all objects seen
- **comparison** [code 2]: specify a list of expected objects
> More on the `sciroc_objdet` README (*WIP*).

1. Once the `/objdet_interface` action server receives a goal, it uses one of its 3 inner action clients (one for each mode) to request the start of the detection process to a specific action server provided by `sciroc_darknet_bridge` (also 3, one for each mode)
2. each one of the action servers of `sciroc_darknet_bridge` behaves similarly after receiving a goal request: it starts sampling the camera topic (possibly asking for the robot head to move in the meantime), sending each frame to `darknet_ros`, again using an action. 
3. `darknet_ros` reads the image received in a goal request to its action server, performs detection on it, and then sends the generated information as an action result.
4. the active server of `sciroc_darknet_bridge` stores each reply, containing the detected boxes and tags, and after the sample phase aggregates the collected data and replies back to the `/objdet_interface`.
5. finally, the aggregated data is sent to `/objdet_interface`, which wraps it back up and sends it as the action result.

## Parameters Editing

To only parameters that should make sense to modify are probably:
-  the camera topic: you can find it under the `sciroc_darknet_bridge/config/objdet.yaml` configuration file, at the `objdet/subscribers/camera_reading/topic`. It's used by the bridge to sample images from the topic where the camera data flow is published.
- detection threshold: `darknet_ros/darknet_ros/config/yolov3.yaml` (which is the default used here), under `yolo_model/threshold/value`.
- head moving Action Server: these packages have been written with Tiago robot from PAL Robotics as the deployment platform. It can, among other things, move its head via an action server, which info are specified in the config file `sciroc_darknet_bridge/config/objdet.yaml`, under `objdet/actions/head_movement`. Parameters there should be self explanatory, but it's worth mentioning that such feature can be disabled by simply setting the parameter `enabled` to false. Try to vary `min_duration` to gather more or less sample images, since the bridge will keep sampling from the moment it receives a request up until the head stopped moving (*notice that if the movement it's not `enable`d this value still indicates the time frame length for the sampling*).
- detection parameters: always at `sciroc_darknet_bridge/config/objdet.yaml`, under `objdet/detection` we have
	- a `period` (*seconds*), which defines the sample period of images from the camera topic to be sent to `darknet_ros` to be analized by YOLO. Notice it's a lower bound to the period (aka upper bound to frequency), since the bridge will autonomously drop frames if the computation from yolo requires longer (+ the overhead from transport).
	- various `threshold`s, generally it's discouraged to specify this, as the aforementioned one under `darknet_ros` provides a more efficient solution. However, these can be fine tuned allowing for different detetion threshold for the three implemented request modes (enumeration, classification, comparison).
	- different `selection_mode`s, which specify how the data from the various sampled images are aggregated together to form the reply to the interface request. We have:
		- `AVG`: in the **enumeration** case, return the average number of objects detected among all frames; for **classification** and **comparison** *to be defined*
		- `MODE`: in the **enumeration** case, return the mode (*most frequent value*) among the number of detected objects in all frames; for **classification** and **comparison** *to be defined*
		- `MAX`: in the **enumeration** case, return the maximum number of detected objects in a single frame, among all sampled frames; for **classification** and **comparison**, return all the objects detected throughout all sample images.
