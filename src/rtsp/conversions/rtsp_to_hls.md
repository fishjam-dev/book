# RTSP to HLS

> Some of the terms used here are explained in detail in the section [RTSP to WebRTC](./rtsp_to_webrtc.md). Refer there if you're lost.

RTSP to HLS conversion is relatively simple. The stream has to be extracted from RTP packets anyway, so there's no point
trying to inject parameters into the stream, it's easier to pass them on externally down the pipeline to the H264 parser.

Keep in mind, however, that (usually) keyframes will still be sent at a fixed rate. Also, it might be in your best interest to
ensure the HLS segment duration overlaps with keyframe frequency.
