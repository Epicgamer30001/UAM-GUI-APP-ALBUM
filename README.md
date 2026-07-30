# UAM-GUI-APP-ALBUM
This repo exists to show my work while the source is still private. It documents a
VLM-powered vision pipeline I built over summer 2026 at SFU's RAMP Lab.

## What it is
A ROS 1 pipeline that turns a stereo camera feed into something you can ask questions
about in natural language. A ZED camera streams RGB and depth, a detection node
produces structured object detections, and an orchestrator packages those detections
into context for a vision-language model. You type a question, the VLM answers it
grounded in what the robot is actually looking at.

![VLM perception pipeline](concept_map.png)

The design goal was modularity: perception, orchestration, and inference
are independent nodes that talk over topics and services, so the VLM backend can be
swapped without touching the detection stack.


## Components
Lets dive into each component of the pipeline.

### Eagle Wrapper Node 

The first component is the Eagle 2.5 VLM wrapper. Eagle 2.5 is NVIDIA's vision-language
model family built for embodied AI, so it handles multi-image and spatial reasoning
better than a general-purpose captioning model.

![Eagle Wrapper Concept Map](eagle_concept_map.png)

The wrapper is split into two scripts:

**`eagle2_5_node.py`** (server) -- loads the model weights into GPU memory once at
  startup and advertises the `vlm/query` service. Model loading takes multiple seconds, so
  keeping it loaded is the whole point of a persistent node rather than a subprocess
  per query. Once a service is requested,, the corresponding json message is corresponded and the VLM is queried. The result is then returned back to the client, and published to   `vlm/answer`. 



**`vlm_client_node.py`** (client) -- subscribes to `vlm/request`, calls `vlm/query`,
  and republishes the result on `vlm/answer`. Isolating the blocking service call here
  means inference latency never stalls the rest of the graph.