# VLM-Augmented Behavior Trees for Autonomous Aerial Object Retrieval under Semantic Uncertainty

*RAMP Lab, Simon Fraser University · Summer 2026*

At SFU's [RAMP Lab](https://www.ramplab.ca/), I've been building the new perception and motion planning stack for aerial object retrieval, powered by NVIDIA's Eagle 2.5 VLM and Ultralytics' YOLO26x. Most recently I've been wrapping all of it into a GUI app, so the same interface drives the system in simulation and on real hardware.

The source is still private, so this repo is the album.

<!-- ASSET 1: the hook. 15-25s looping GIF of the GUI driving a full retrieval.
     You have full-stack demo footage -- cut the tightest window where the
     operator does something and the drone responds. GIF, not a video link.
     Nobody clicks the link. -->
![Full system demo](media/demo.gif)

[Full video demo →](github.com)

---

## The platform

<!-- ASSET 2: the real hardware photo, not Gazebo. Gazebo moves to the sim
     section. If the arm and both cameras are visible, add labelled arrows
     matching the table. -->
![UAM platform](uam_gazebo.png)

- **AureliaX6 Pro (UAM Platform)** -- hexacopter airframe that carries the rest of the system
- **Open-Manipulator-X (4 DOF Arm)** -- robotic arm mounted underneath the hexacopter
- **Gripper (End Effector)** -- two-fingered gripper at the end of the arm
- **ETH Camera (Eye-to-Hand)** -- fixed camera on the body, viewing the arm/workspace from outside
- **EIH Camera (Eye-in-Hand)** -- camera mounted near the gripper, co-moving with it for closed-loop manipulation

<!-- ASSET 3: you have this one. ETH and EIH views of the same instant,
     side by side. Makes the distinction obvious in one glance. -->
![ETH and EIH views](real_uam.png)

---


<!-- ASSET 4: one diagram showing all four layers stacked, with the arrows
     between them. This is the load-bearing figure of the whole README --
     it is the map a reader keeps coming back to. Worth an hour in draw.io.
     concept_map.png currently only covers the perception layer. -->

## Three layers

![System architecture](media/system_architecture.png)

**Perception.** ZED stereo feeds YOLO26x, whose 2D bounding boxes get lifted into 3D against the registered depth map, and the result is packaged as scene context that a VLM can answer questions about.

**Motion planning.** Behavior tree communicates with the perception stack and issues commands to the high-level controller, originally authored by [Mohammad Soltanshah](https://msoltanshah.github.io/) and later redesigned by me, which in turn drives the low-level controller. That bottom layer is Abolfazl Eskandarpour's [MPC](https://www.researchgate.net/publication/364582025_A_Constrained_Robust_Switching_MPC_Structure_for_Tilt-Rotor_UAVs_Trajectory_Tracking_Problem), which handles the coupled arm-rotor dynamics and sends the actual rotor and arm commands to the vehicle.

**GUI.** This is a tabbed user interface wrapping all of the above, with both camera feeds, a chatbot-style VLM query interface, and live arm control, running against simulation and hardware through the same interface.

Motion Planning and the GUI app is currently in progress.

# Under the hood

## Perception

![Perception pipeline](concept_map.png)

| Node | Owner | Rate | Output |
|---|---|---|---|
| `zed_wrapper` | StereoLabs | 30 Hz | Rectified RGB, registered depth, intrinsics |
| `detection_node.py` | mine | ~15 Hz (RTX 4090) | 3D detections, JSON scene, annotated frame |
| `orchestrator_node.py` | mine | on demand | Assembled prompt + grounding frame |
| `vlm_client_node.py` | mine | on demand | Blocking service call |
| `eagle2_5_node.py` | mine | ~`[X]` s / query | Answer string |

### One query, end to end

<!-- ASSET 9: pair your terminal Q&A screenshot with the annotated detection
     frame from the SAME moment, stacked into one two-panel image. The objects
     named in the answer must be the ones boxed in the frame. -->
![Worked example](media/worked_example.png)

What the detector handed the orchestrator for that frame:

```json
[
  {"class": "bottle", "conf": 0.91, "bbox": [412, 233, 486, 388], "position": [0.31, -0.08, 0.74]},
  {"class": "cup",    "conf": 0.87, "bbox": [198, 261, 254, 331], "position": [-0.22, 0.04, 0.61]}
]
```

What Eagle 2.5 returned:

```
[paste the actual answer string]
```

End to end: **`[X.X]`** s, of which **`[X.X]`** s is Eagle inference. Weight load at startup is **`[X]`** s, which is the entire reason the model lives in a persistent node rather than a subprocess per query.

The model is not being asked to caption a photo. It is told what the detector found, where each object sits in metric 3D, and how confident the detector was.

### ZED node

Provided by StereoLabs as part of the [ZED ROS wrapper](https://www.stereolabs.com/docs/ros/). Depth is computed on-board via stereo matching and republished as standard ROS topics.

| Topic | Type | Purpose |
|---|---|---|
| `rgb/image_rect_color` | `sensor_msgs/Image` | Rectified colour frames |
| `depth/depth_registered` | `sensor_msgs/Image` | Per-pixel depth, pixel-aligned to RGB |
| `rgb/camera_info` | `sensor_msgs/CameraInfo` | Intrinsics |

Registered depth is the one that matters: it shares a frame with the colour image, so a pixel inside a detection box maps straight to a depth reading with no extra transform.

<!-- ASSET 10: 30-second capture. rqt_image_view on both topics, one screenshot.
     Shows the alignment instead of asserting it. -->
![RGB and registered depth](media/rgb_depth_pair.png)

### `detection_node.py`

![Object detection concept map](object_detection/object_detection_concept_map.png)

YOLO26x produces 2D boxes on each RGB frame. Each box is lifted into 3D: sample the registered depth across the box region, take the interquartile range as the object's depth to reject background bleed at the box edges, and back-project through the intrinsics for a metric position. ~15 Hz on an RTX 4090.

<!-- ASSET 11: the IQR trick is the most interesting decision in this node and
     right now it is one clause of one sentence. A depth histogram inside one
     bounding box -- object peak, background tail, IQR window drawn on -- is
     the figure people would remember from this repo. Matplotlib, ~20 lines. -->
![Depth sampling within a bounding box](media/depth_iqr.png)

<!-- ASSET 12: you have live RViz footage. Use a short GIF of detections
     tracking a moving object rather than the static screenshot -- motion
     shows stability, a still cannot. -->
![RViz detections](object_detection/rviz_detections.png)

| Topic | Type | Purpose |
|---|---|---|
| `object_detection/detections` | `vision_msgs/Detection3DArray` | Structured 3D detections for the BT layer |
| `object_detection/detections_json` | `std_msgs/String` | Same content serialized for prompt injection |
| `object_detection/scene` | `[type]` | `[what this carries beyond detections]` |
| `object_detection/image` | `sensor_msgs/Image` | Annotated frame, OpenCV-drawn |
| `object_detection/markers` | `visualization_msgs/MarkerArray` | 3D boxes for RViz |

### `orchestrator_node.py`

Holds the most recent scene state from `detections_json` and `scene`, waits on `vlm/question`, and on arrival assembles a prompt from the question plus current detections, publishing it on `vlm/request` alongside the frame the answer should be grounded in.

The prompt template:

```
[paste it -- this is the actual contribution of the node, and right now the
 README describes it instead of showing it]
```

Scene state older than `[X]` ms is `[flagged / refused / used anyway with a warning]`.

### Eagle 2.5 wrapper

![Eagle wrapper concept map](eagle/eagle_concept_map.png)

![Eagle wrapper demo](eagle/eagle_demo.png)

Eagle 2.5 is NVIDIA's VLM family built for embodied AI, so it handles multi-image and spatial reasoning better than a general-purpose captioning model. The wrapper turns it into a ROS service any node can call.

**`eagle2_5_node.py`** (server) loads weights into GPU memory once at startup, advertises `vlm/query`, runs inference on request, returns the answer to the caller, and republishes it on `vlm/answer`.

**`vlm_client_node.py`** (client) subscribes to `vlm/request`, packs the message into a `vlm/query` request, and blocks on the call.

| `vlm/query` field | Type | Purpose |
|---|---|---|
| `images` | `sensor_msgs/Image[]` | Frames passed inline, typically straight off the ZED |
| `image_paths` | `string[]` | Alternative to `images`, loads from disk |
| `prompt` | `string` | Assembled prompt from the orchestrator |
| `media_type` | `string` | `text` / `image` / `multi_image` / `video` |
| `id` | `uint32` | Correlates response with request (optional) |

### Design decisions

**`vlm/answer` is published by the server, not the client.** Requests arrive two ways: via the `vlm/request` topic, or by calling `vlm/query` directly. Publishing from the server catches both. Publishing from the client would silently drop every direct service call.

**The blocking call lives in its own node.** ROS 1 service calls are synchronous and inference takes seconds. Confining that call to `vlm_client_node.py` means only that process stalls; the detector and orchestrator keep spinning at full rate.

**Detections are published twice, typed and as JSON.** The typed message is what the behavior tree consumes. The JSON copy exists because the orchestrator feeds it into a text prompt, and serializing once here beats every consumer writing its own formatter.

**Prompt construction lives in the orchestrator, not the client.** The client stays a dumb relay, so rephrasing scene context never touches the transport layer and the VLM backend stays swappable.

**Visualization topics are separate from data topics.** `object_detection/image` and `markers` cost bandwidth and only serve RViz, so nothing in the pipeline depends on them and both can be left unsubscribed in deployment.

---

## Behavior tree framework

<!-- ASSET 13: a rendered tree. py_trees_ros can dump the live tree, or
     draw the retrieval tree by hand. Colour the nodes by tick status
     mid-run if you can capture that -- a static tree is a diagram, a
     ticking tree is a demo. -->
![Retrieval behavior tree](media/bt_retrieval.png)

`[The retrieval behavior, described as the tree reads: search, approach, align,
grasp, retreat, or whatever the actual decomposition is.]`

`[What "modular" buys concretely. The strongest version of this claim is a second
tree built from the same leaves doing a different task, shown side by side with
the first. If that exists, it is the best figure in this section.]`

| Node | Type | Purpose |
|---|---|---|
| `[LeafName]` | `[Behaviour / ReactiveFallback / Sequence]` | `[what it does]` |

### Design decisions

`[Why a behavior tree over a state machine here. The answer is usually reactivity
and composability, but say it in terms of a specific failure this system had.]`

`[How leaves talk to ROS: service calls, action clients, blackboard? What happens
to a tick while a leaf is waiting on the VLM for several seconds?]`

---

## High-level commander

<!-- ASSET 14: the before/after is the whole story of a rewrite. Two
     diagrams, old structure and new, side by side. -->
![Commander redesign](media/commander_redesign.png)

`[What the original commander was, and the specific thing that made it need
replacing. "Redesigned" means nothing to a reader without the problem statement.]`

**What changed:**

| | Before | After |
|---|---|---|
| `[axis]` | `[old]` | `[new]` |

`[Measurable outcome if one exists: lines removed, states collapsed, latency,
a class of bug that stopped happening.]`

---

## Simulation

![Gazebo](uam_gazebo.png)

`[If you developed against Gazebo before flying hardware, say so. A sim-vs-real
pair of the same retrieval attempt shows a workflow, not just a result.]`

---

## Status

- [x] Perception stack: YOLO26x detection with metric 3D lifting
- [x] Eagle 2.5 wrapper, service and topic interfaces
- [x] Orchestrator prompt assembly
- [x] Modular behavior tree framework
- [x] High-level commander redesign
- [x] GUI: camera, VLM query, and arm control tabs
- [ ] `[next tab]`
- [ ] `[next milestone]`

Built at RAMP Lab, Simon Fraser University, under `[supervisor credit]`.