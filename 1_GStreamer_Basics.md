# GStreamer Basics
Nvidea DeepStream is built using [GStreamer](https://github.com/GStreamer/gstreamer), which is a library for constructing graphs of media-handling components. Familiarity with some key aspects of GStreamer makes working with DeepStream far easier.

At its core, a GStreamer program is a *graph of elements that move buffers from one element to the next*. GStreamer will handle scheduling, memory allocation, synchronisation, and threading. 

## GStreamer Core
These are some core GStreamer concepts which are useful when writing Nvidea DeepStream pipelines.

### Buffers
A `GstBuffer` is a chunk of data. It moves through the pipeline, created by sources, and consumed by sinks. Buffers flow downstream through the pipeline, whereas Events (EOS, Flush, Seek) flow upstream.

### Elements
Elements are processing units which perform some work on a Gst Buffer. Each Gst Element has:
- Properties
- Pads
- Caps

An Element may:
- transform buffers (decoder, converter)
- change metadata (nvtracker)
- produce new buffers (sources)
- not modify any buffers at all (queue)

### Pads
Elements communicate using Pads, which connect elements. Pads are the defined connection points on an Gst Element where data enters or leaves. Functioning like a physical port on an piece of equipment.
- Determine the direction of data flow, and the type of data.
- Connection between two elements if formed by linking an 'output pad' (or 'source pad') of one element to an 'input pad' or ('sink pad') of the next element.
	- Example source pad: decoder element's source pads outputs raw video frames.
	- Example sink pad: encoder element's sink pad accepts raw video frames for compression.

Some Pads are static, and some are request pads (meaning created dyamically). DeepStream uses request Pads.
```python
element.get_static_pad("src")
element.get_request_pad("sink_0")
```

Pads are where you can attach:
- Pad probes
- Metadata hooks
- Buffer inspection

### Caps
Caps describe allowed formats. They are defined on Elements so that elements can understand each other when linked. Negotiation is normally automatic unless a capsfilter forces a specific format.

Example:
- video/x-h264
- video/x-raw
- video/x-raw(memory:NVMM)

### Linking Elements
GStreamer Elements can be linked using two main methods. 

An **Easy Link** is possible where:
- `a` has a src pad
- `b` has a sink pad
- Caps automatically match for `a` and `b`

This looks like :
```python
a.link(b)
```

**Manual Linking** is required when:
- Pads are created dynamically
- Caps need negotiation

An Example:
```python
decoder_src_pad = decoder.get_static_pad("src")
streammux_sink_pad = streammux.get_request_pad("sink_0")
decoder_src_pad.link(streammux_sink_pad)
```


### The Bus
The Bus is a messaging channel from pipeline elements back to the application. It reports:
- Errors
- Warnings
- End-of-stream
- State changes

Without a bus handler, the app may appear to 'hang'. 

### Pipeline
A pipeline is an [[1_GStreamer_Basics#Elements|Element]] which contains other elements. It manages:
- State changes
- Clocking
- Bus Messages
- Resource cleanup
### Main loop
GStreamer requires an asynchronous event loop to;
- Process bus messages
- Schedule frames
- Handle async pads
- Keep the application alive.

```python
loop = GLib.MainLoop()
loop.run()
```

### Element Lifecycle
Every element goes through these states:
- NULL → unloaded
- READY → allocated
- PAUSED → preroll (buffers arriving but not playing)
- PLAYING → realtime streaming

Transition is invoked by:
```python
pipeline.set_state(Gst.State.PLAYING)
```

## Code
This example shows some barebones code for a purely GStreamer pipeline.

```python
import gi
gi.require_version('Gst', '1.0')
from gi.repository import Gst, GLib

Gst.init(None)


def bus_call(bus, message, loop):
    t = message.type
    if t == Gst.MessageType.EOS:
        print("End of stream")
        loop.quit()
    elif t == Gst.MessageType.ERROR:
        err, debug = message.parse_error()
        print(f"Error: {err}, {debug}")
        loop.quit()
    return True


def main():
    pipeline = Gst.Pipeline()

    # Create elements
    source = Gst.ElementFactory.make("filesrc", "file-source")
    h264parse = Gst.ElementFactory.make("h264parse", "h264-parse")
    decoder = Gst.ElementFactory.make("nvv4l2decoder", "decoder")
    sink = Gst.ElementFactory.make("fakesink", "fake-sink")

    if not all([pipeline, source, h264parse, decoder, sink]):
        print("ERROR: Could not create all GStreamer elements")
        return

    # Configure elements
    source.set_property("location", "input.mp4")

    # Add to pipeline
    pipeline.add(source)
    pipeline.add(h264parse)
    pipeline.add(decoder)
    pipeline.add(sink)

    # Link elements
    source.link(h264parse)
    h264parse.link(decoder)
    decoder.link(sink)

    # Main loop
    loop = GLib.MainLoop()
    bus = pipeline.get_bus()
    bus.add_signal_watch()
    bus.connect("message", bus_call, loop)

    print("Starting pipeline")
    pipeline.set_state(Gst.State.PLAYING)

    try:
        loop.run()
    except KeyboardInterrupt:
        pass

    print("Exiting")
    pipeline.set_state(Gst.State.NULL)


if __name__ == '__main__':
    main()

```