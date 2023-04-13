# Actually using the stream
_What use is a stream if one cannot view it?_

When we were working with RTSP, we attempted to:
1. deliver the stream to a WebRTC peer, and
2. convert the received stream to an HLS playlist.

Unfortunately, to decode the stream, we must first deal with several issues.
