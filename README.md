# capture-and-streaming
Blocks for capturing video, processing it, and streaming it over WebRTC. (Note: This is still a work in progress. Once completed it will be moved to a different Org. Do not rely on this repo while it is still in the Playground.)

## Usage
The capture block will automatically set `BALENA_HOST_CONFIG_start_x` to `1` and `BALENA_HOST_CONFIG_gpu_mem` to `192`. This effectively utilizes 192 MB of system memory for the GPU, regardless of your dashboard setting for "Define device GPU memory in megabytes." Note that when the block changes these settings, your device will reboot. (Usually, this only happens the first time the block runs.)

### Capture Block

The Capture Block takes a video source (usually a camera) as an input and converts it to an RTSP stream. It utilizes [gst-rtsp-server](https://github.com/GStreamer/gst-rtsp-server) for core functionality.

Create the Device Variable `GST_V4L2_USE_LIBV4L2` to `1` to enable the camera.

Input: The block will search for a Pi Camera (currently only on a Raspberry Pi) and use that by default. If it does not find a Pi Camera, it will look for a USB camera and use the first one it finds. If the camera supports YUYV it will use that, otherwise it will use mjpeg. You can override this automatic selection process by specifying your own Gstreamer pipeline using the service variable `GST_RTSP_PIPELINE`. 

You can also use a custom pipeline to change the default height, width, and framerate. For instance, here is the default pipeline used when a Pi camera is detected:

`rpicamsrc bitrate=8000000 awb-mode=tungsten preview=false ! video/x-h264, width=640, height=480, framerate=30/1 ! h264parse ! rtph264pay name=pay0 pt=96`

To change the height and width when using a pi Camera, edit the values in the above pipeline and then set the `GST_RTSP_PIPELINE` to the edited pipeline.

For a standard USB webcam that uses YUYV:

`v4l2src device=/dev/video0 !  v4l2convert ! video/x-raw,width=640,height=480,framerate=15/1 ! omxh264enc target-bitrate=6000000 control-rate=variable ! video/x-h264,profile=baseline ! rtph264pay name=pay0 pt=96`

On the Jetson Nano it would be:

`v4l2src device=/dev/video0 ! nvvidconv ! video/x-raw(memory:NVMM) ! omxh264enc bitrate=6000000 control-rate=variable ! video/x-h264,profile=baseline ! rtph264pay name=pay0 pt=96`

For an older webcam that only supports jpeg:

`v4l2src device=/dev/video0 ! image/jpeg,width=640,height=480,framerate=10/1 ! queue ! jpegdec ! omxh264enc target-bitrate=6000000 control-rate=variable ! video/x-h264,profile=baseline ! rtph264pay name=pay0 pt=96`

In fact, you can change almost any part of the pipeline to suit your needs, but make sure it uses the proper device name and ends with an rtp output such as `rtph264pay`. Also be sure that your camera (or source) supports the desired size and framerate. When the capture block starts, it displays this information. You can also ssh into the block and run `v4l2-ctl --list-formats-ext` to see supported features.

A RTSP stream will be available on `rtsp://localhost:8554/server` to other containers in the application. (Replace localhost with the device's IP address to view the stream outside the device)

### Streaming Block

The Streaming Block takes an RTSP stream as an input and produces a WebRTC stream as an output. The input stream is selected automatically but can be overriden. If a processing block is running on the device, it will use that block's RTSP output stream as an input. If no processing block is found, it will look for a capture block and use that as the input. If neither are found, or you want to override this behavior, you can specify an RTSP input stream with the service variable `WEBRTC_RTSP_INPUT`. By default, the output WebRTC stream will be on port 80, but you can change that by specifying the service variable `WEBRTC_PORT`.

The streaming block utilizes [webrtc-streamer](https://github.com/mpromonet/webrtc-streamer) so all of its features are available to use as well. For instance, you can pass the options shown in [this list](https://github.com/mpromonet/webrtc-streamer#usage) using the `WEBRTC_OPTIONS` device variable. It must be used in conjunction with `WEBRTC_RTSP_INPUT` to create a full input for the webrtc-streamer. If you supply a value for `WEBRTC_OPTIONS` it will launch the following command, giving you full control over the streamer:
```
./webrtc-streamer WEBRTC_RTSP_INPUT WEBRTC_OPTIONS
```

So for instance if you want to specify an external TURN server, use:

`WEBRTC_RTSP_INPUT` = `-H 0.0.0.0:80` and `WEBRTC_OPTIONS` = `-sstun.l.google.com:19302` which is equivalent to:

```
./webrtc-streamer -H 0.0.0.0:80 -sstun.l.google.com:19302
```


### Processing Block
The processing block (coming soon!) allows you to transform an RTSP stream. It will automatically use the capture block as its input if it exists on the device. Otherwise, specify an RTSP input stream with the `PROC_RTSP_INPUT` service variable. The output of the processing block will be available on `rtsp://localhost:8558/proc` to other containers in the application. (Replace localhost with the device's IP address to view the stream outside the device.) This block is ideally suited for use with the capture and streaming blocks, and video will automatically be routed through this block if the other two are present.

An API is used to control the processing block. The block is written in python using OpenCV and it's relatively easy to add additional functionality. The base API supports:
```
/api/process
/api/flip
/api/mirror
/api/rotate
/api/negate
/api/monochrome
/api/text
```

