#  On Screen Display
The Gst-nvdsosd plugin draws bounding boxes, text, arrows, lines, circles, and regions of interest ROI polygons. These can be drawn on the frames and then reconstructed in the output stream.

## Code

```python
# Setting up nvdsosd in the pipeline
nvdsosd = Gst.ElementFactory.make("nvdsosd", "onscreendisplay")
pipeline.add(nvdsosd)
nvvideoconvert.link(nvdsosd)
nvdsosd.link(sink)
```

Typically we would retrieve the sink pad of the nvdsosd element, and then add a 'pad probe' to the sink pad, such that we can inspect each buffer passing through the element. This is discussed in [Metadata](2_Metadata.md#Accessing%20or%20Modifying%20Metadata|Accessing%20or%20Modifying%20Metadata).

```python
osdsinkpad = nvdsosd.get_static_pad("sink")
osdsinkpad.add_probe(Gst.PadProbeType.BUFFER, osd_sink_pad_buffer_probe, 0)
```

Referring to the `osd_sink_pad_buffer_probe` function example in [Metadata](2_Metadata.md#Accessing%20or%20Modifying%20Metadata|Accessing%20or%20Modifying%20Metadata), we can update the parameters of the display.

```python
def osd_sink_pad_buffer_probe(pad: Gst.Pad, info: Gst.PadProbeInfo, u_data: Any):
	...
	while l_frame is not None:
		try:
		...
		
		while l_obj is not None:
		    try:
			...
			
			# Now we can access and set aspects of the display.for the objects.
			txt_params = obj_meta.text_params
			txt_params.display_text = "Example"
			txt_params.font_params.font_name = "Serif"
```