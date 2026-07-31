# UAM-GUI-APP-ALBUM
This repo exists to show my work while the source is still private. It documents a
VLM-powered vision pipeline I built over summer 2026 at SFU's RAMP Lab.
[Here](github.com) is a video demo of the full stack system. 

## What it is
A ROS 1 pipeline that turns a stereo camera feed into something you can ask questions
about in natural language. A ZED camera streams RGB and depth, a detection node
produces structured object detections, and an orchestrator packages those detections
into context for a vision-language model. You type a question, the VLM answers it
grounded in what the robot is actually looking at.

### perception pipline:
![VLM perception pipeline](concept_map.png)

The design goal was modularity: perception, orchestration, and inference
are independent nodes that talk over topics and services, so the VLM backend can be
swapped without touching the detection stack.


## Components
Let us dive into each component of the pipeline.

### Eagle Wrapper Node 
The first component is the Eagle 2.5 VLM wrapper. Eagle 2.5 is NVIDIA's vision-language
model family built for embodied AI, so it handles multi-image and spatial reasoning
better than a general-purpose captioning model.
![Eagle Wrapper Demo](eagle/eagle_demo.png)

![Eagle Wrapper Concept Map](eagle/eagle_concept_map.png)

The wrapper is split into two scripts:

**`eagle2_5_node.py`** (server) -- loads the model weights into GPU memory once at
startup and advertises the `vlm/query` service. Loading takes several seconds, which is
the whole reason this is a persistent node rather than a subprocess spawned per query.
On each request the node parses the incoming fields, runs inference, returns the answer
string to the caller, and publishes the same result on `vlm/answer` for any other node
that wants to listen.


**`vlm_client_node.py`** (client) -- subscribes to `vlm/request`, packs the message into
a `vlm/query` service request, and calls the server. 

**vlm/query custom message**
| Field | Type | Purpose |
|---|---|---|
| `images` | `sensor_msgs/Image[]` | Frames passed inline, typically straight off the ZED |
| `image_paths` | `string[]` | Alternative to `images`, loads from disk instead |
| `prompt` | `string` | The assembled prompt from the orchestrator |
| `media_type` | `string` | Selects the input mode: `[text/image/multi_image/video]` |
| `id` | `uint32` | Correlates a response with its request (optional) |

#### Architectural decisions

**`vlm/answer` is published by the server, not the client.** Requests can arrive two ways:
via the `vlm/request` topic, or by calling `vlm/query` directly. Publishing from the server
means `vlm/answer` catches both. Publishing from the client would miss direct service calls.

**The blocking call lives in its own node.** ROS 1 service calls are synchronous, and VLM
inference takes multiple seconds. Confining that call to `vlm_client_node.py` means only it stalls
`detection_node.py` and `orchestrator_node.py` are separate processes and keep spinning.


### Detection Pipeline

Three nodes sit between the camera and the VLM: the ZED driver, `detection_node.py`, and
`orchestrator_node.py`. Together they turn raw stereo frames into a text description of
the scene that a language model can reason over.

#### ZED node

Provided by StereoLabs as part of the [ZED ROS wrapper](https://www.stereolabs.com/docs/ros/).
The camera computes depth on-board via stereo matching, and the wrapper republishes it as
standard ROS topics. Nothing in this repo touches the SDK directly -- the driver is
launched from `[launch file]` and everything downstream just subscribes.

The three topics that matter here:

| Topic | Type | Purpose |
|---|---|---|
| `rgb/image_rect_color` | `sensor_msgs/Image` | Rectified colour frames |
| `depth/depth_registered` | `sensor_msgs/Image` | Per-pixel depth, pixel-aligned to the RGB frame |
| `rgb/camera_info` | `sensor_msgs/CameraInfo` | Intrinsics |

Registered depth is the important one: because it shares a frame with the colour image, a
pixel in a detection box maps directly to a depth reading with no extra transform.

#### `detection_node.py`

Runs [YOLO variant] on each RGB frame to get 2D boxes, then lifts each box into 3D. For a
given box it samples the depth image over the box region, takes [the median? centroid?] to
reject outliers, and back-projects through the camera intrinsics to get a position in
`[frame_id]`. Output runs at roughly [N] Hz on [GPU].

Publishes:

| Topic | Type | Purpose |
|---|---|---|
| `object_detection/detections` | `[vision_msgs/Detection3DArray?]` | Structured detections for downstream nodes |
| `object_detection/detections_json` | `std_msgs/String` | Same content serialized for the orchestrator |
| `object_detection/scene` | `[type]` | [What this carries beyond detections] |
| `object_detection/image` | `sensor_msgs/Image` | Annotated frame for RViz |
| `object_detection/markers` | `visualization_msgs/MarkerArray` | 3D boxes for RViz |

#### `orchestrator_node.py`

The glue between perception and inference. It holds the most recent scene state from
`detections_json` and `scene`, and waits on `vlm/question` for user input. When a question
arrives it builds a prompt combining the question with the current detections -- object
labels, positions, [confidences?] -- and publishes it on `vlm/request` along with the
frame the answer should be grounded in.

This is what makes answers scene-specific rather than generic captioning. The model is not
just shown an image; it is told what the detector found and where.

[Worth adding: the actual prompt template, and how stale a scene is allowed to be before
the orchestrator refuses or flags it.]

#### Architectural decisions

**Detections are published twice, typed and as JSON.** The typed message is what a planner
or controller should consume. The JSON copy exists because the orchestrator feeds it into a
text prompt, and serializing once here beats every consumer writing its own formatter.

**Prompt construction lives in the orchestrator, not the client.** `vlm_client_node.py`
stays a dumb relay, so changing how scene context is phrased never touches the transport
layer, and the VLM backend can be swapped without rewriting prompt logic.

**Visualization topics are separate from data topics.** `object_detection/image` and
`markers` cost bandwidth and only serve RViz, so nothing in the pipeline depends on them
and they can be left unsubscribed in deployment.