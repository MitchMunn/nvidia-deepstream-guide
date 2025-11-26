# Inference 
Inference can be done in one of two ways;
1. TensorRT
2. Triton

## TensorRT
The Gst-nvinfer plugin accepts batched Nv12/RGBA buffers from upstream.  The plugin has three modes of operation.

**Modes**:
1. Primary mode: operates on full frames
2. Secondary mode: operates on objects added in the metadata by upstream components
3. Preprocessed Tensor input mode: operates on tensors attached by upstream components.

In primary and secondary mode, the plugin performs transforms such as format conversion an scaling on the input based on the network requirements, and passes the transformed data to the low level library. The low level library preprocesses the transformed frames by performing normalization and mean subtraction, which are then passed to the TensorRT inference engine.

In preprocessed tensor input mode, the pre-processing inside Gst-nvinfer is completely skipped. The plugin will pass the tensor from the input buffer directly to the TensorRT inference function without any modifications.
### Code
```python
pgie = GST.ElementFactory.make("nvinfer", 'nvinfer-plugin")
pgie.set_property("config-file-path", "path")
pgie.set_property("input-tensor-meta", 1)
pipeline.add(pgie)

streammux.link(pgie)
pgie.link(tracker)
```

### Config
```yaml
[property]
# General behaviour of the plugin
...

[class-attrs-all]
# Detection parameters for all classes
...

[class-attrs-<id>]
# Detection parameters for a single class
...
```
