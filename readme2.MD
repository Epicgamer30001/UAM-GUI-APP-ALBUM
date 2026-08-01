# UAM-GUI-APP-ALBUM

**Ask an aerial manipulator what it's looking at, and get an answer grounded in its own sensors.**

A ROS 1 Noetic perception stack built over summer 2026 at SFU's [RAMP Lab](https://www.ramplab.ca/), pairing a YOLO26x + ZED depth detector with NVIDIA's Eagle 2.5 vision-language model. Source is still private; this repo is the album.

<!-- ASSET 1: 10-20s looping GIF, autoplaying, above everything else.
     Ideal shot: split screen -- annotated detection feed on one side,
     terminal showing question typed and answer streaming back on the other.
     A GIF beats a video link. Nobody clicks the link. -->
![Full stack demo](media/demo.gif)

[Full video demo →](github.com)

---

## It answers questions about what it sees

<!-- ASSET 2: the single most valuable image in this README.
     A screenshot of one real query end to end. Ideally a 3-panel figure:
     (a) the raw scene photo, (b) the annotated frame with boxes + depth labels,
     (c) the terminal with the question and Eagle's answer. -->
![Worked example](media/worked_example.png)

```
$ rostopic pub /vlm/question std_msgs/String "data: 'Which object is closest to the gripper?'"
```

What the detector handed the orchestrator for that frame:

```json
[
  {"class": "bottle", "conf": 0.91, "bbox": [412, 233, 486, 388], "position": [0.31, -0.08, 0.74]},
  {"class": "cup",    "conf": 0.87, "bbox": [198, 261, 254, 331], "position": [-0.22, 0.04, 0.61]}
]
```

What Eagle 2.5 returned:

```
[paste the actual answer string here]
```

Latency, end to end: **[X.X] s** -- **[X.X] s** of that is Eagle inference, the rest is prompt assembly and transport.

The point of the JSON above is that the model is not being asked to caption a photo. It is told what the detector found, where each object is in metric 3D, and how confident the detector was. Ask the same question of raw Eagle 2.5 with no scene context and you get **[what you actually get -- a vague answer? a hallucinated object? show it]**.

---

## The platform

<!-- ASSET 3: you have uam_gazebo.png, but a photo of the REAL hardware
     on the bench beats a sim render for a showcase repo. If you have both,
     use the photo here and keep the Gazebo shot in the sim section below. -->
![UAM platform](media/uam_hardware.png)

| | |
|---|---|
| **AureliaX6 Pro** | Hexacopter airframe carrying the full stack |
| **Open-Manipulator-X** | 4 DOF arm mounted underneath |
| **Two-finger gripper** | End effector |
| **ETH camera** | Body-fixed, views arm and workspace from outside |
| **EIH camera** | Near the gripper, co-moves with it for closed-loop manipulation |

<!-- ASSET 4: side-by-side ETH view vs EIH view of the same moment.
     This makes the eye-to-hand / eye-in-hand distinction obvious in one glance
     instead of two bullet points. -->
![ETH and EIH views](media/eth_eih_comparison.png)

---

## Pipeline

![Perception pipeline](concept_map.png)

| Node | Owner | Rate | Output |
|---|---|---|---|
| `zed_wrapper` | StereoLabs | 30 Hz | Rectified RGB, registered depth, intrinsics |
| `detection_node.py` | mine | ~15 Hz (RTX 4090) | 3D detections, JSON scene, annotated frame |
| `orchestrator_node.py` | mine | on demand | Assembled prompt + grounding frame |
| `vlm_client_node.py` | mine | on demand | Blocking service call |
| `eagle2_5_node.py` | mine | ~[X] s / query | Answer string |

Perception, orchestration, and inference are separate processes talking over topics and services. Concretely, that meant swapping the VLM backend from **[what you swapped from]** to Eagle 2.5 touched **[N]** files and zero lines of the detection stack.

---

## Eagle 2.5 wrapper

Eagle 2.5 is NVIDIA's VLM family built for embodied AI, so it handles multi-image and spatial reasoning better than a general-purpose captioning model. The wrapper turns it into a ROS service any node can call.

![Eagle wrapper concept map](eagle/eagle_concept_map.png)

![Eagle wrapper demo](eagle/eagle_demo.png)

**`eagle2_5_node.py`** (server) loads weights into GPU memory once at startup -- **[X]** s, which is the entire reason this is a persistent node and not a subprocess per query. It advertises `vlm/query`, runs inference on request, returns the answer to the caller, and republishes it on `vlm/answer`.

**`vlm_client_node.py`** (client) subscribes to `vlm/request`, packs the message into a `vlm/query` request, and blocks on the call.

### `vlm/query` service definition

| Field | Type | Purpose |
|---|---|---|
| `images` | `sensor_msgs/Image[]` | Frames passed inline, typically straight off the ZED |
| `image_paths` | `string[]` | Alternative to `images`, loads from disk |
| `prompt` | `string` | Assembled prompt from the orchestrator |
| `media_type` | `string` | `text` / `image` / `multi_image` / `video` |
| `id` | `uint32` | Correlates response with request (optional) |

### Design decisions

**`vlm/answer` is published by the server, not the client.** Requests arrive two ways: via the `vlm/request` topic, or by calling `vlm/query` directly. Publishing from the server catches both. Publishing from the client would silently drop every direct service call.

**The blocking call lives in its own node.** ROS 1 service calls are synchronous and inference takes seconds. Confining that call to `vlm_client_node.py` means only that process stalls -- `detection_node.py` and `orchestrator_node.py` keep spinning at full rate. **[Worth showing: `rostopic hz /object_detection/detections` during an in-flight query, proving the detector never drops.]**

---

## Detection pipeline

![Object detection concept map](object_detection/object_detection_concept_map.png)

### ZED node

Provided by StereoLabs as part of the [ZED ROS wrapper](https://www.stereolabs.com/docs/ros/). Depth is computed on-board via stereo matching and republished as standard ROS topics.

| Topic | Type | Purpose |
|---|---|---|
| `rgb/image_rect_color` | `sensor_msgs/Image` | Rectified colour frames |
| `depth/depth_registered` | `sensor_msgs/Image` | Per-pixel depth, pixel-aligned to RGB |
| `rgb/camera_info` | `sensor_msgs/CameraInfo` | Intrinsics |

Registered depth is the one that matters: it shares a frame with the colour image, so a pixel inside a detection box maps straight to a depth reading with no extra transform.

<!-- ASSET 5: RGB frame and its registered depth map side by side, same instant.
     Shows the alignment claim instead of asserting it. -->
![RGB and registered depth](media/rgb_depth_pair.png)

### `detection_node.py`

YOLO26x produces 2D boxes on each RGB frame. Each box is then lifted into 3D: sample the registered depth over the box region, take the interquartile range as the object's depth to reject background bleed at the box edges, and back-project through the intrinsics for a metric position. ~15 Hz on an RTX 4090.

<!-- ASSET 6: the IQR trick is the most interesting thing in this node and it's
     currently one clause of one sentence. A figure showing the depth histogram
     inside one bounding box -- object peak vs background tail, with the IQR
     window drawn on -- turns it into the thing people remember about this repo. -->
![Depth sampling within a bounding box](media/depth_iqr.png)

Taking the naive box-center depth instead puts the object at **[X.XX] m** when ground truth is **[X.XX] m**, because **[the centre pixel lands on a gap / the handle / etc.]**.

![RViz detections](object_detection/rviz_detections.png)

| Topic | Type | Purpose |
|---|---|---|
| `object_detection/detections` | `vision_msgs/Detection3DArray` | Structured 3D detections for planners and controllers |
| `object_detection/detections_json` | `std_msgs/String` | Same content serialized for prompt injection |
| `object_detection/scene` | `[type]` | `[what this carries beyond detections]` |
| `object_detection/image` | `sensor_msgs/Image` | Annotated frame, OpenCV-drawn |
| `object_detection/markers` | `visualization_msgs/MarkerArray` | 3D boxes for RViz |

### `orchestrator_node.py`

Holds the most recent scene state from `detections_json` and `scene`, waits on `vlm/question`, and on arrival assembles a prompt from the question plus current detections, publishing it on `vlm/request` alongside the frame the answer should be grounded in.

The prompt template:

```
[paste it -- this is the actual contribution of the node and right now
 the README describes it instead of showing it]
```

Scene state older than **[X]** ms is **[flagged / refused / used anyway with a warning]**, because **[reason]**.

### Design decisions

**Detections are published twice, typed and as JSON.** The typed message is what a planner or controller consumes. The JSON copy exists because the orchestrator feeds it into a text prompt, and serializing once here beats every consumer writing its own formatter.

**Prompt construction lives in the orchestrator, not the client.** `vlm_client_node.py` stays a dumb relay, so rephrasing scene context never touches the transport layer and the VLM backend stays swappable.

**Visualization topics are separate from data topics.** `object_detection/image` and `markers` cost bandwidth and only serve RViz, so nothing in the pipeline depends on them and both can be left unsubscribed in deployment. Measured saving: **[X]** MB/s.

---

## Simulation

![Gazebo](uam_gazebo.png)

**[If you developed against Gazebo before hardware, say so and show a sim-vs-real
pair of the same task. It demonstrates a workflow, not just a result.]**

---

## Status

- [x] Eagle 2.5 wrapper, service + topic interfaces
- [x] YOLO26x detection with metric 3D lifting
- [x] Orchestrator prompt assembly
- [ ] **[next thing]**
- [ ] **[next thing]**

Built at RAMP Lab, Simon Fraser University, under **[supervisor credit]**.