# Batching 
NVIDEA Deepstream will batch different sources (ie multiple cameras) such that we have:
- A single buffer
- With multiple frames
- Each frame containing potentially multiple objects

This is critical so that subsequent processes such as inference can do multiple frames simultaneously. This batching process is performed by the ***GST-nvstreammux plugin***. A Multiplexer (MUX) takes several input buffers from the sources and can synthesize a single output signal.

In order for *GST-nvstreammux* to operate, it is unique in that it needs to dynamically create a new input pad for every source you want to connect.  *sink_%u* is the required pattern for dynamically doing this, where 'sink' identifies an input pad, and '%u' is a format specifier that gets replaced by an index number.

The typical process:
```python
# Create the streammux element
streammux = Gst.ElementFactory.make("nvstreammux", "Stream-muxer")

# request a pad on the muxer element
# of pad type 'sink', replace 'u' with an int
sinkpad = gst_element_get_request_pad("sink_%u")
```

This will create your sinkpad object which ACCEPTS data. 

## Code
Now that you have a sinkpad for your streammux plugin, you  would then want to link this to an output pad object, such that you now have data flowing from your output pad (such as the decoder) to the input pad of the streammux object.
```python
streammux = Gst.ElementFactory.make("nvstreammux", "nvstreammux-plugin")
streammux.set_property("config-file-path", "path")
pipeline.add(streammux)

sinkpad = gst_element_get_request_pad("sink_%u")
srcpad = convert.get_static_pad("src") # assume preceding element is 'convert'

srcpad.link(sinkpad)
```

## Waiting for frames
With N sources, each batch meta object will consist of a list of frames, of size N\*batch_size. The mux will 'wait' for each of the sources to produce a buffer and make it available to streamux, before pushing the output batched buffer downstream. This is crucial since each source may have different FPS, and different capture latencies. 
- If the batch is 'filled' (ie all sources have produced a frame), the output buffer is immediately pushed downstream.
- If the *batch formation timeout* is reached, the output buffer is pushed downstream, regardless of missing source buffers. Contributing factors for this timeout are:
	- Overall FPS
	- Individual stream FPS
  The timeout starts running when the first buffer for a new batch is collected
  - Batches use a 'round-robin' algorithm to collect batches from their sources.
	  - Tries to collect an average of batch-size/num-source frames per batch from each source (if all frame rates are the same)
	

## Memory Management
- The muxer forwards the frames from the source (without copying the data) into a pre-allocated output buffer, which is large enough to contain each frame from each source.
- When the muxer passes its batched output buffer downstream, the muxer temporarily 'loses ownership' of that memory.
- When the entire batched buffer has completed the downstream component and is no longer needed, the last element in the pipeline returns the batched buffer to the muxer.
- The muxer then releases that frame memory associated with each stream, signalling the corresponding source that the memory space where its original frame was stored is free and available to use.
