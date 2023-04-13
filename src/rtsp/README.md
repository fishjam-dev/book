# RTSP

RTSP (Real Time Streaming Protocol) is an application-level protocol used to control the delivery of data with real-time properties.
It was defined in [RFC 2326](https://www.rfc-editor.org/rfc/rfc2326). It's mainly used in media servers and surveillance web cameras.

> Note that [RFC 7826](https://www.rfc-editor.org/rfc/rfc7826) defines RTSP 2.0, which makes the previous document obsolete,
> but this chapter will only refer to RTSP 1.0, as that's the version of the protocol we worked with.

It uses the same concepts as basic HTTP, though an important distinction between these two is that RTSP is not stateless.

RTSP by itself provides only the means to **control** the delivery of the stream. The actual stream contents are delivered
using mechanisms based upon RTP (Real-time Transport Protocol). This means that RTCP (RTP Control Protocol) messages
may also be exchanged between the server and the client using the media delivery channel (more on that in the following sections).

While working with RTSP, we encountered quite a few setbacks and roadblocks,
which we're going to describe in detail in this chapter.
