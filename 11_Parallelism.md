# Parallelism 
Often DeepStream pipelines require GStreamer elements to run in parallel. One such example of this is when you want to publish message payloads over a network as well as send your video to sink. In this example, we want to 'branch' the pipeline into two so that the independent processes of publishing messages and sending video can occur decoupled, preventing either from slowing down the other.

## GStreamer Basics
### Tee
A Tee is a GStreamer element which splits a single stream into multiple branches.
- It has a single sink pad
- It has N requestable src_pads (`src_%u`) 
- It duplicates the buffer references without copying the data, and pushes the same buffer to each branch

### Queue
A Queue is a GStreamer element that is a bounded FIFO buffer plus a streaming thread boundary. A queue decouples the upstream from the downstream. The upstream pushes into the queue, and then a separate thread pulls and processes the downstream.  A queue will create a new streaming thread per queue. 


## Code
By creating multiple branches (src pads) using a Tee element, and attaching a separate queue to each src pad (one per branch), each of the branches will run concurrently in its own thread. With multiple CPU cores, these thread may execute in parallel. This is shown in the example below, where messages are published to a broker while simultaneously the video is outputted.

```python
tee = Gst.ElementFactory.make("tee", "tee")
queue1 = Gst.ElementFactory.make("queue", "queue1")
queue2 = Gst.ElementFactory.make("queue", "queue2")

pipeline.add(tee)
pipeline.add(queue1)
pipeline.add(queue2)
pipeline.add(msgconv)
pipeline.add(msgbroker)
pipeline.add(sink)

# previous_element is some upstream element like nvosd
previous_element.link(tee)

# BRANCH 1 -> publish messages
queue_1_sink_pad = queue1.get_static_pad("sink")
tee_msg_pad = tee.request_pad_simple('src_%u')
tee_msg_pad.link(queue_1_sink_pad)
queue1.link(msgconv)
msgconv.link(msgbroker)

# BRANCH 2 -> output video
queue_2_sink_pad = queue2.get_static_pad("sink")
tee_render_pad = tee.request_pad_simple('src_%u')
tee_render_pad.link(queue_2_sink_pad)
queue2.link(sink)

```

## Improvements
In real-time, we often want to make branches tolerant to stalls. We can set various properties on the Gst Queue element to accommodate this.

For example, for the messaging broker, we can make the 'max-size-buffers' small to make an upper limit for how many buffers the queue can hold. We can also set the 'leaky' property, which controls what happens when the queue is full, to 'downstream', which drops the oldest buffer already in the queue to make room for a new one.
```python
queue1.set_property('max-size-buffers', 20)
queue1.set_property('leaky', 2) # downstream
```
The result of these settings is bounded latency with possible skips.

Likewise, for the video stream, we can set the 'max-size-time' to 2 seconds, providing a cap on the total buffered duration. Setting 'leaky' to 0 will block upstream while the downstream consumes, which preserves all data but increases latency.

```python
queue2.set_property('max-size-time', 2000000000) # 2s
queue2.set_property('leaky', 0) # None
```

**Be Careful** because Tee creates references to the underlying object, not copies. Therefore, any changes made in one of the branches to the buffer will also be seen in other branches. Therefore, care must be taken to not mutate the shared buffer in ways that can affect other branches.