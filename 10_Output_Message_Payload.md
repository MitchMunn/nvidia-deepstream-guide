# Output Message Payload

In DeepStream, outputting a message payload can generally be split into three main components.
  - Attach message intent: add NvDsEventMsgMeta (optional if using new API).
  - Serialize: nvmsgconv converts to JSON and attaches NvDsPayloadMeta.
  - Publish: nvmsgbroker reads NvDsPayloadMeta and sends via the selected protocol adapter

First, a `NVDS_EVENT_MSG_META` type must be added to the Metadata on the Buffer. This can be done by attaching a pad probe at one of the Gst Element sink pads, and attaching the message metadata. Next,  the plugin `gst-nvmsgconv` is responsible for converting the `NVDS_EVENT_MSG_META` payload into JSON format, which is attached as a `NVDS_PAYLOAD_META` type metadata onto the buffer. Finally `nvmsgbroker` plugin extracts the `NVDS_PAYLOAD_META` type of metadata from the metadata ands sends the payload to the server.

## Metadata
###  `NvDsEventMsgMeta`
The `NvDsEventMsgMeta` [API reference](https://docs.nvidia.com/metropolis/deepstream/dev-guide/python-api/PYTHON_API/NvDsMetaSchema/NvDsEventMsgMeta.htm) indicates the possible information that the class can contain, which are common fields. We see that typical fields refer to:
- Detection information (bbox, classID)
- Sensor information
- Time and position information
- Tracking information

Notably, various types of custom objects can be attached to the message. These are not common fields - generally specific to the particular use case of the deployment. DeepStream ships with a number of built in custom objects which can be found in the [API reference](https://docs.nvidia.com/metropolis/deepstream/dev-guide/python-api/PYTHON_API/NvDsMetaSchema/NvDsObjectType.htm), some of which are 'VEHICLE', 'PERSON', 'FACE' etc. These custom objects are attached through using the `extMsg` and `extMsgSize` fields on the `NvDsEventMsgMeta` object.

### Code
Referring to the `osd_sink_pad_buffer_probe` function example in [Metadata](2_Metadata.md#Accessing%20or%20Modifying%20Metadata|Accessing%20or%20Modifying%20Metadata), we can attach the `NvDsEventMsgMeta` to the Metadata. In this example,  we e detecting vehicles and so use the built in `NvDsVehicleObject` custom type.

```python
def osd_sink_pad_buffer_probe(pad: Gst.Pad, info: Gst.PadProbeInfo, u_data: Any):
	...
	while l_frame is not None:
		try:
		...
		
		while l_obj is not None:
		    try:
			...
			
			# Get an empty metadata block that can be filled
			user_event_meta = pyds.nvds_acquire_user_meta_from_pool(batch_meta)
			
			# Allocate NvDsEventMsgMeta structure
			# Using alloc means the underlying memory is not managed by Python.
			# Ensures downstream plugins can access this data, avoiding falling out of scope
			msg_meta: NvDsEventMsgMeta = pyds.alloc_nvds_event_msg_meta(user_event_meta)

			# Now we need to set the fields in the NvDsEventMsgMeta object
			msg_meta.bbox.top = obj_meta.rect_params.top
			msg_meta.bbox.left = obj_meta.rect_params.left
			msg_meta.bbox.width = obj_meta.rect_params.width
			msg_meta.bbox.height = obj_meta.rect_params.height
			msg_meta.frameId = frame_number
			msg_meta.trackingId = long_to_uint64(obj_meta.object_id)
			msg_meta.confidence = obj_meta.confidence
			msg_meta.sensorId = 0
			msg_meta.placeId = 0
			msg_meta.moduleId = 0
			msg_meta.sensorStr = "sensor-0"
			msg_meta.ts = pyds.alloc_buffer(MAX_TIME_STAMP_LEN + 1)
			pyds.generate_ts_rfc3339(msg_meta.ts, MAX_TIME_STAMP_LEN)
			
			# Object Specific fields
			msg_meta.type = pyds.NvDsEventType.NVDS_EVENT_MOVING
	        msg_meta.objType = pyds.NvDsObjectType.NVDS_OBJECT_TYPE_VEHICLE
	        msg_meta.objClassId = PGIE_CLASS_ID_VEHICLE
	        obj = pyds.alloc_nvds_vehicle_object()
			# Hard code object fields, normally would grab from inference.
			obj.type = "sedan"
			obj.color = "blue"
			obj.make = "Bugatti"
			obj.model = "M"
			obj.license = "XX1234"
			obj.region = "C"
	        msg_meta.extMsg = obj
	        msg_meta.extMsgSize = sys.getsizeof(pyds.NvDsVehicleObject)

			# Attach metadata to the frame, making it available to downstream elements
			user_event_meta.user_meta_data = msg_meta
			user_event_meta.base_meta.meta_type = pyds.NvDsMetaType.NVDS_EVENT_MSG_MET
			pyds.nvds_add_user_meta_to_frame(frame_meta, user_event_meta)
```


### What if your classes aren't supported by existing Nvidia Object Types?
NVidia has an Enum `NvDsObjectType` which has the supported objects, such as 'VEHICLE', 'PERSON' etc. If these are not suitable for your usecase, there are two main options:

**Option 1**: use an 'UNKNOWN' type.
If you don't actually care about messaging 'more' information about the class itself outside of just its class ID, there is no need to set the extMsg field with a pointer to your Class (such as `NvDsVehicleObject`). Hence, we can set:
```python
meta.objType = pyds.NvDsObjectType.NVDS_OBJECT_TYPE_UNKNOWN
meta.objClassId = <your detector class ID>
```
This way, `nvmsgconv` will serialise it as 'unknown'.

**Option 2**: Attach a custom JSON blob
 Using the “new API” path so `nvmsgconv` parses `NvDsFrameMeta`/`NvDsObjectMeta` and can also include a user-provided JSON blob. This is achieved by:
  - Set msgconv.set_property('msg2p-newapi', True) and choose your schema (payload-type=1 is typical for compact payloads).
  - At runtime, build a small JSON object with whatever fields you need and attach it as user metadata on the NvDsFrameMeta.

**Option 3**: Extend the payload generator to support a new object (C/C++)
This option involves teaching the underlying `nvmsgconv` library how to serialize your custom object type. This involves:
  - Define a new C struct (e.g., `NvDsMyObject`) representing your custom fields.
  - Extend the payload generator (the library used by gst-nvmsgconv) to recognize and serialize your type to JSON.
  - Rebuild and deploy the updated payload generator library alongside DeepStream.
## Serialization

### Gst-nvmsgconv
The Gst-nvmsgconv plugin is responsible for converting metadata into message payloads based on schema.  The generated payload (ie, the serialisation of the `NVDS_EVENT_MSG_META` into JSON)  is attached back into the input buffer as `NVDS_PAYLOAD_META` type user metadata.

#### Control Parameters
**payload-type**: an integer specifying whether to use the 'full' schema (payload-type=0), or the 'minimal' schema (payload-type=1).
- Full Schema: one payload per object, includes rich fields (essential fields + optional extMsg)
- Minimal Schema: one payload can include multiple objects per frame, and contains only essential fields.

**msg2p-lib**: a boolean, where 'True' builds the payload using existing frame/object meta, and 'False' builds the payload using event-message meta.
- Event-message meta: this is where you attach `NVDS_EVENT_MSG_META` as user metadata on each frame/object, from which `nvmsgconv` reads these and builds the payloads. This is the default, legacy approach that affords you far greater control.
- Existing frame/object meta: A newer approach is available where nvmsgconv directly parses the existing `NVDS_FRAME_META` and `NVDS_OBJECT_META` that DeepSteam already has to build the payload, avoiding the need to create `NvDsEventMsgMeta` yourself. However, you lose control over which objects, and when payloads are created.

### Code
```python
msgconv = Gst.ElementFactory.make("nvmsgconv", "nvmsg-converter")
pipeline.add(msgconv)

previous_element.link(msgconv)
msgconv.link(msgbroker)
```

## Send Payload
The `Gst-nvmsgbroker` plugin sends payload messages to the server using a specified communication protocol. It accepts Gst Buffers that have a `NvDsPayload` metadata attached. Messages are sent to the server using the `nvdsmsgapi` interface, and the particular protocol you want to use must implement this interface. The following protocols are shipped with DeepStream out of the box:
- MQTT
- Kafka
- Azure MQTT
- AMQP
- REDIS
- MQTT

The protocol adapters provided with DeepStream can be found at  `/opt/nvidia/deepstream/deepstream/sources/libs/*_protocol_adaptor`

The `nvds_msgapi` supports functions such as:
- Creating a connection
- Sending messages by synchronous or asynchronous means
- Terminating the connection
- Coordinating the client’s and protocol adapter’s use of CPU resources and threads
- Getting the protocol adapter’s version number

Note that messages are only sent 
### Code
```python
msgbroker = Gst.ElementFactory.make("nvmsgbroker", "msg_broker")
pipeline.add(msgbroker)
msgconv.link(msgbroker)
```

### Configuration

```yaml
proto-lib: path to adapter, e.g., /opt/nvidia/deepstream/deepstream/lib/libnvds_kafka_proto.so
conn-str: backend connection string
config: optional adapter config file
topic: optional topic override
sync: False to avoid blocking the branch on network latency
```