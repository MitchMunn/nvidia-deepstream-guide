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

Typically we would retrieve the sink pad of the nvdsosd element, and then add a 'pad probe' to the sink pad, such that we can inspect each buffer passing through the element. This is discussed in [[2_Metadata#Accessing or Modifying Metadata|Accessing or Modifying Metadata]].

```python
osdsinkpad = nvdsosd.get_static_pad("sink")
osdsinkpad.add_probe(Gst.PadProbeType.BUFFER, osd_sink_pad_buffer_probe, 0)
```

Within the `osd_sink_pad_buffer_probe` function at the `NvDsObjectMeta` level, we can update the parameters of the display.

```python
# Now we can access and set aspects of the display.
txt_params = obj_meta.text_params
txt_params.display_text = "Example"
txt_params.font_params.font_name = "Serif"
```