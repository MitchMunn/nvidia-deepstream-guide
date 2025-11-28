# Video Source 
AIM: get video from the sensor into NVMM (Nvidia memory) for maximum performance. These pipelines ensure that:
1. Video arrives properly
2. It is decoded WITH NVIDIA hardware.
3. It is provided to DeepStream in the correct format (NV12, NVMM)

Note that NV12 is a type of video-optimized compressed format that saves space by separating luminance and chrominance information. RGB is better if you want to conserve color fidelity.

A typical edge video enabled device requires the following steps: Input, Prepare for Decoder, Decode.
## Input
**Input**: collecting video bytes from a camera, file, or network stream. In GStreamer we use elements to do this:
- *Examples*
	- nvarguscamerasrc: for a CSI Camera (Jetson).  The raw video frames are captured.
	- rtspsrc: RTSP (Real Time Streaming Protocol) Network camera. Outputs raw compressed data stream encapsulated in transport packed (like RTP).
- *Output:*
	- GST Buffer: a container which holds  video frame data.
	- The physical address of where the video frame data is in memory depends on the element used. On Jetson, the data is stored in NVMM, which is typically device memory accessible to the multimedia hardware accelerators (ISP, VIC, GPU, encoders/decoders) without needing to go through the main CPU.

## Prepare for Decoder
**Prepare for Decoder**: parse, depay, convert format. The raw data retrieved by the source element is prepared and reshaped into a standard format that a hardware or software decoder can understand.
- *Depacketization*: or *Depay*, necessary for a network stream (like RTSP), where the video data is broken down into packets for transport. It reassembles the RTP packets, stripping away the RTP header information to output raw H.264 elementary stream data.
	- Example: rtph264depay
	- H.264: a compressed video format.
- *Parse*: necessary to handle compressed video formats (h.264), which are complex bitstreams that contain structural information (sequence parameter sets, picture parameter sets) that the decoder needs. It essentially analyses the structure of the data to ensure headers are packed with the corresponding frame data, making it fully compliant.
	- Example: h264parse
- *Format Conversion*: If data is uncompressed video frame (such as from nvarguscamerasrc), this element converts the color space or frame layout to a format preferred by the decoder.
	- Input: uncompressed video frame (e.g RAW, YUYV, RGB format)
	- Output: typically NV12 color format.
	- Example: nvvidconv
	It can also be used to convert data that has been decoded back into raw frames.
## Decode
**Decode**: turn compressed video (H.264, MJPEG) into raw frame (NVMM GPU memory).
- Examples: nvv4l2decoder (H264 hardware decode), nvjpegdec (JPEG hardware decode)

## Examples

**VSI camera (on Jetson)**
```bash
nvarguscamerasrc ! nvvidconv
```
- nvarguscamerasrc: onnect to NVIDIA ISP (image signal processor)
- nvvidconv: convert camera output format

**Regular USB webcams**
```bash
v412src ! nvjpegdec
```
- v412src: reads video from a USB camera or any V4L2 device
- nvjpegdec: hardware decoder for MJPEG

**RTSP camera**
```bash
rtspsrc ! rtph264depay ! h264parse ! nvv4l2decoder
```

**Video file (H264)**
```bash
filesrc ! h264parse ! nvv4l2decoder
```

## Code

```python
source = Gst.ElementFactory.make("nvarguscamerasrc", "nvarguscamerasrc-plugin")
pipeline.add(source)
convert = Gst.ElementFactory.make("nvvidconv", "nvvidconv-plugin")
pipeline.add(convert)
```

We would then use the `Gst-nvstreammux` plugin to move the our source into a batch in the pipeline, as shown [here](4_Batching.md).