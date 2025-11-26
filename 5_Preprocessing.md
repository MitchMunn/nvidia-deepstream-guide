# Preprocessing 
The GST-nvdspreprocess plugin is for preprocessing on input frames. Each stream can have its own preprocessing requirements, with streams with the same preprocessing requirements grouped and processed together.

**Functionalities:**
- Streams with predefined ROIs (Regions of Interest) are scaled and converted as per the network requirements for inference.
- Raw tensor created from scaled and converted ROI, passed to downstream plugins on the batched buffer.

**Modes:**
- PGIE: process the given ROI/Frame for primary inferencing.
- SGIE: process the detected object within the ROI/Frame for secondary inferencing.

The default functionalities are provided by the default custom library (nvdspreprocess_lib). However, other libraries are available or you can write your own. These libraries can be specified in the config.

## Code
The following code sets up the preprocessing plugin within the DeepStream pipeline.
```python
preprocess = Gst.ElementFactory.make("nvdsdpreprocess", "preprocess-plugin")
preprocess.set_property("config-file", "config_preprocess.txt")
pipeline.add(preprocess)
streammux.link(preprocess)
preprocess.link(pgie)
```


## Configuration
Configuration for the preprocessing plugin is split into 'property', 'user-configs' and 'group'.
```yaml
[property]
# General behaviour of the plugin
...

[user-configs]
# Parameters required by the custom library
...

[group-<id>]
# Parameters specific to individual streams
```


## Tiling
Tiling is a common process in computer vision object detection pipelines where the object of interest can be small relative to the frames of the source. In order to improve performance, the frame is 'tiled' or 'subdivided' into smaller frames which are each passed to the inference engine. 

We can use the library ''libcustom2d_preprocess.so" to perform tiling.

Since we want preprocessing to occur within the preprocessing plugin, we need to tell deepstream nvinfer plugin (which by default will perform its own preprocessing) to NOT perform an preprocessing. We do this by telling nvinfer that the input is already preprocessed.
```python
pgie.set_property("input-tensor-meta", True)
```

### Configuration
Here is some of the configuration required to enable tiling (not full config!).

```yaml
[property]
network-input-order=0 # NCHW
network-color-format=0 # RGB
# In this case, N=2, since Two ROIs are defined, and we are using the custom library
# C=3, since color-format is RGB
network-input-shape=2:3:640:640
# Stating the library being used
custom-lib-path=/opt/nvidia/deepstream/deepstream/lib/gst-plugins/libcustom2d_preprocess.so
custom-tensor-preparation-function=CustomTensorPreparation
...

[group-0] # Call this 'group 0'
src-ids=0 # apply preprocessing to src with id 0 -> ie 'sink_0'
custom-input-transformation-function=CustomAsyncTransformation
process-on-roi=1 # process on the ROIs, not the full frame
# Two ROIs: (left; top; width; height)
roi-params-src-0=0;0;640;640;640;0;640;640
```

### Source
[Applying inference over specific frame regions with nvidia deepstream](https://developer.nvidia.com/blog/applying-inference-over-specific-frame-regions-with-nvidia-deepstream/)
