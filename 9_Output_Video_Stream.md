# TODO: Output Stream 

## Code
```python
convert_out = GST.ElementFactory.make("nvvidconv", "convert_out")
encoder = GST.ElementFactory.make("nvv4l2h264enc", "h264-encoder")
rtp_payloader = GST.ElementFactory.make("rtph264pay", "rtp-payloader")
sink = GST.ElementFactory.make("fakesink", "fakesink") # TODO: use RTSP server

nvosd.link(convert_out)
convert_out.link(encoder)
encoder.link(rtp_payloader)
rtp_payloader.link(sink)
```