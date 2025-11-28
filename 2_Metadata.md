# Metadata 
A Gst Buffer is the basic unit of data transfer in GStreamer, where each Buffer has associated metadata. Deepstream attaches metadata to the Buffer throughout various plugins within the pipeline. Metadata generally exists at three levels in a nested format:
1. Batch-Level: a 'batch' consists of one or more frames. These frames may be from a single, or multiple camera sources.
2. Frame-level: a single frame may consist of one or more 'objects' (ie detections), and display params for those objects.
3. Object-level: an object is a grouping of information relating to a detection in a frame.

We see this structure below:

![deepstream_metadata](deepstream_metadata.png)
User-specific data can be attached at any of the batch, frame, or object level (see [](2_Metadata.md#Attaching%20User%20Metadata|Attaching%20User%20Metadata)). This is useful for attaching additional metadata specific to your application.

- `NvDsBatchMeta` : the basic metadata structure at the 'batch level'. Attached within the `Gst-nvdsstreammux` plugin.
- `NvDsFrameMeta`: metadata at the 'frame level'. Created in `nvstreammux` and populated by the primary inference plugin (PGIE) such as `nvinfer`, which will create the `NvDsFrameMeta`, and add it into the `NvDsBatchMeta.NvDsFrameMetalist`.
- `NvDsObjectmeta`: metadata at the 'object level'. Created after PGIE run inference on each frame when it parses detection. Each detected object is allocated and attached to `NvDsFrameMeta.object_meta_list`.

## Accessing or Modifying Metadata
Metadata can be accessed or modified at specific points in the deepstream pipeline. From the previous section, we know that DeepStream Metadata is attached to the GstBuffer. In the GStreamer  (and hence the DeepStream pipeline), the GstBuffer flows through pipeline Gst Pads.

Metadata can be accessed or modified only *inside a pad probe* or *inside a custom GStreamer plugin*. Typically (but not exclusively) this is done in:
- The sink pad of `nvinfer`, to read frame data before inference or modify/add metadata before PGIE runs.
- The src pad of `nvinfer`, to read inference output, modify `NvDsObjectMeta`.
- The src pad of `nvtracker`, to access tracking IDs, update object metadata with tracking info.
- Any pad of a custom plugin

Note that we previously mentioned that `NvDsBatchMeta` is created in the `Gst-nvdsstreammux` plugin. The [documentation](https://docs.nvidia.com/metropolis/deepstream/dev-guide/text/DS_plugin_metadata.html#adding-custom-meta-in-gst-plugins-upstream-from-gst-nvstreammu) notes that it is possible to add metadata *before* the `Gst-nvdsstreammux` plugin.

### Pad Probes
Pad probes can be attached to Gst Pads in the DeepStream Pipeline. In the example below, we get the sink pad for the `nvdsosd` Gst Element in the pipeline, and attach a probe to it. Now the probe function is called on every buffer. 

```python
nvdsosd = Gst.ElementFactory.make("nvdsosd", "nvdsosd-plugin")
osdsinkpad = nvosd.get_static_pad("sink")
osdsinkpad.add_probe(Gst.PadProbeType.BUFFER, osd_sink_pad_buffer_probe, 0)
```

We can define our pad probe function. Obviously the video pipeline runs in C/CUDA on GPU memory. Python cannot own or copy that memory efficiently. We will see that Python gets lightweight wrappers that let it read and modify the C metadata in place, without taking ownership from C, copying, or allowing the data to be garbage collected in Python.

```python
def osd_sink_pad_buffer_probe(pad: Gst.Pad, info: Gst.PadProbeInfo, u_data: Any):
""" A pad probe callback which accesses and inspects each buffer passing through the sinkpad.

Args:
	pad: the sink pad of the nvdsosd element.
	info:  a GStreamer struct from which the Gst buffer containing the video frame and deepstream metadata can be passed.
	u_data: optional user data of Any type
"""

	# Access the gst_buffer
	gst_buffer = info.get_buffer()
	if not gst_buffer:
		return Gst.PadProbeReturn.OK
	
	# Retrieve batch metadata from the gst_buffer
	# Pass the C address of the gst_buffer, obtained with hash
	batch_meta: pyds.NvDsBatchMeta = pyds.gst_buffer_get_nvds_batch_meta(hash(gst_buffer))
	
	if not batch_meta:
		return Gst.PadProbeReturn.OK
		
	# frame_meta_list is a C linked list exposed through Python Bindings
	# l_frame is a pointer to one node in the linked list frame_meta_list
	l_frame: pyds.NvDsLinkedList = batch_meta.frame_meta_list
	while l_frame is not None:
		try:
			# DeepStream metadata lives in C, not Python. To safely access, we use 'cast'	
			# which: 
				# - wraps the C struct 'l_frame.data' of type 'NvDsFrameMeta' in Python obj
				# - does not copy the data
				# - Python will not free the memory, C owns it
			frame_meta = pyds.NvDsFrameMeta.cast(l_frame.data)
		except StopIteration:
			# Defensively wrap traversal of linked list so pipeline does not crash
			continue
	
		# obj_meta_list is a pointer to a C linked list 'NvDsObjectMetalist'
		l_obj: pyds.NvDsObjectMetaList = frame_meta.obj_meta_list
		while l_obj is not None:
		    try:
				# Same Pattern as previous
		        obj_meta = pyds.NvDsObjectMeta.cast(l_obj.data)
		    except StopIteration:
		        continue
				
			# Now we can Access or modify the metadata!
			# In this example, lets just set aspects of the display.
			txt_params = obj_meta.text_params
			txt_params.display_text = "Example"
			txt_params.font_params.font_name = "Serif"
			
			try:
				# Advance the pointer in the linked list to grab the next frame
                l_obj = l_obj.next
            except StopIteration:
                break
				
		try:
            l_frame = l_frame.next
        except StopIteration:
            break
	
	return Gst.PadProbeReturn.OK
```

One thing to note is that `probe()` callbacks are synchronous and thus holds the Gst buffer from traversing the pipeline until the user return. This is likely more costly in Python rather than C.

## Attaching User Metadata
User specific metadata can be attached to the `NvDsBatchMeta` within a Gst Buffer at the batch, frame, object level.
Referencing the code in [](2_Metadata.md#Accessing%20or%20Modifying%20Metadata|Accessing%20or%20Modifying%20Metadata). This is done by acquiring a user_meta object and setting some required fields in the object.

```python
# Creating the NvDsUserMeta object
user_meta: NvDsUserMeta = nvds_acquire_user_meta_from_pool(batch_meta)
user_meta.user_meta_data = <any_python_object>
user_meta.base_meta.meta_type = <unique_int>   # typically pyds.NvDsMetaType.NVDS_USER_META

# Copy and release functions to safely duplicate metadata internally
user_meta.copy_user_meta_func = <copy_func>
user_meta.release_user_meta_func = <free_func>

# Adding the NvDsUserMeta object at the desired batch, frame, or object level
pyds.nvds_add_user_meta_to_frame(frame_meta, user_meta) # frame level here.
```

For a concrete example, we may use:
```python
import pyds

def my_copy_meta_func(data, user_data):
    meta = pyds.NvDsUserMeta.cast(data)
	
	# returning a Python object is fine ONLY if user_meta_data does not contain 
	# GPU pointers, device memory, or external C structs.
	# Generally dicts, strings, ints, JSON, pure Python types are safe.
    return meta.user_meta_data

def my_release_meta_func(data, user_data):
    meta = pyds.NvDsUserMeta.cast(data)
    meta.user_meta_data = None
    return
	
	
def osd_sink_pad_buffer_probe(pad: Gst.Pad, info: Gst.PadProbeInfo, u_data: Any):
	... # This Code is incomplete - Reference previous code for osd_sink_probe func
	
    # ---- BATCH-LEVEL CUSTOM META ----
    batch_user_meta = pyds.nvds_acquire_user_meta_from_pool(batch_meta)
	batch_user_meta.user_meta_data = {"batch_info": "batch1", "note": "example"}

    batch_user_meta.base_meta.meta_type = pyds.NvDsMetaType.NVDS_USER_META
    batch_user_meta.copy_user_meta_func = my_copy_meta_func
    batch_user_meta.release_user_meta_func = my_release_meta_func

    pyds.nvds_add_user_meta_to_batch(batch_meta, batch_user_meta)
	
	while l_frame:
        frame_meta = pyds.NvDsFrameMeta.cast(l_frame.data)

        # ---- FRAME-LEVEL CUSTOM META ----
        user_meta = pyds.nvds_acquire_user_meta_from_pool(batch_meta)

        # add whatever Python object you want
        user_meta.user_meta_data = {"frame_id": frame_meta.frame_num, "note": "example"}
        user_meta.base_meta.meta_type = pyds.NvDsMetaType.NVDS_USER_META
        user_meta.copy_user_meta_func = my_copy_meta_func
        user_meta.release_user_meta_func = my_release_meta_func
        pyds.nvds_add_user_meta_to_frame(frame_meta, user_meta)
		
        # ---- OBJECT-LEVEL CUSTOM META ----
        l_obj = frame_meta.obj_meta_list

        while l_obj:
            obj_meta = pyds.NvDsObjectMeta.cast(l_obj.data)

            user_meta = pyds.nvds_acquire_user_meta_from_pool(batch_meta)
	        user_meta.user_meta_data = {"obj_id": "obj1", "note": "example"}
            user_meta.base_meta.meta_type = pyds.NvDsMetaType.NVDS_USER_META
            user_meta.copy_user_meta_func = my_copy_meta_func
            user_meta.release_user_meta_func = my_release_meta_func

            pyds.nvds_add_user_meta_to_obj(obj_meta, user_meta)
```
