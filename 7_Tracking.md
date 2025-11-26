# Tracking 
The Gst-nvtracker plugin allows the DeepStream pipeline to use the low-level tracker library to track detected objects over time persistently with unique IDs. Included are reference implementations for:
- IOU
- NvSORT
- NvDCF
- MaskTracker
- NvDeepSORT

The plugin queries the low-level library for capabilities and requirements concerning input format, memory type, which the plugin uses to convert the input frame buffers into the format requested.  Each low level library will attach miscellaneous data to the GST buffer which have useful information for tracking such as past-frame object data, targets, trajectory etc. These are can be retrieved by the GST-nvtracker plugin.

## Sub Batching
By default, input frame batch is passed and processed by a single instance of the low-level tracker library. However, if parts of the tracker library run on the CPU, we could have 'GPU idling'. Sub batching is a way to avoid GPU idling by allowing parallel processing in sub-batches.

## Config
```yaml
[tracker]
tracker-width=640
tracker-height=640
gpu-id=0
ll-lib-file=<path-to-tracker-model> # /opt/nvidia/deepstream/deepstream/lib/libnvds_nvmultiobjecttracker.so
ll-config-file=<path-to-tracker-lib.yaml>
```

## Code
```python
tracker = Gst.ElementFactory.make("nvtracker", "nvtracker-plugin")
pipeline.add(tracker)
pgie.link(tracker)
tracker.link(nvvidconv)
```
