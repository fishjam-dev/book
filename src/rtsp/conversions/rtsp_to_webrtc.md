# RTSP to WebRTC

Both RTSP and WebRTC use RTP for stream transportation. This may lead you to think it'd be enough to simply forward
the received RTP packets to WebRTC peers, perhaps changing their SSRC in the process. _How hard can it be?_

Sadly, it's not as easy as it might seem.

## Keyframes
If a new WebRTC peer joins an existing room, we will need to request keyframes (IDR frames) from all other participants.
This poses a problem: RTSP itself does not provide a way to request that a keyframe be transmitted in the RTP stream.

> The issues we write about in this section could maybe be alleviated by sending an RTCP FIR (Full Intra Request),
> RTCP PLI (Picture Loss Indication) or RTCP NACK (Negative Acknowledgement) for the lost packets.
> We won't write about this solution, as not all RTSP servers may support such requests.
> Refer to [RFC 4585](https://www.rfc-editor.org/rfc/rfc4585), [RFC 5104](https://www.rfc-editor.org/rfc/rfc5104)
> and [RFC 6642](https://www.rfc-editor.org/rfc/rfc6642) for more details.

Usually, keyframe frequency may only be configured server-side (globally for all clients) in encoder settings.
The server will send a keyframe to each recipient every *n* frames, and that's about it.

The implications? Real-life example: We had an RTSP camera set up (default settings) so that it had a framerate of 25 fps and transmitted an IDR frame
every 32nd frame, so every 1.28 seconds. This meant that, worst case scenario, you connected to a stream and had to wait over
a second before being able to decode the stream, with no way to shorten this delay other than changing the server's encoder settings.

What's more: Once an IDR frame is lost, it's lost. You can't really do anything on the client side, which means no retransmissions
(unless you use RTP over TCP or the server handles RTCP NACK/PLI).

## Stream parameters
You may know that in order to decode a video stream encoded using H264, you need stream parameters, SPS (Sequence Parameter Set)
and PPS (Picture Parameter Set). These may be transmitted in-band (together with the video stream, contained inside RTP
packets) - if that's the case, no problems here. However, they may also be transmitted out-of-band, meaning they're absent
in the video stream, and have to be delivered in some other way (and that's precisely what most RTSP servers will be doing).

Usually, RTSP servers will transmit stream parameters (together with other useful info) inside a response to the DESCRIBE request.
They will be encoded using SDP (Session Description Protocol, [RFC 8866](https://www.rfc-editor.org/rfc/rfc8866),
parameters: [RFC 6871](https://www.rfc-editor.org/rfc/rfc6871)).

When it comes to WebRTC, the parameters **must** be present in the stream. This means that one needs to parse the parameters from the response,
then somehow include them within the stream itself.

_I'm sure injecting them won't cause any significant problems..._

### Injecting parameters into the stream
Should you wish to include the parameters in the RTP stream more than once (perhaps adding them before every keyframe,
so that new peers are able to decode the stream if they joined later on), you immediately run into issues regarding packet numbering. 

Suppose we have the following RTP packets being sent by the server:
```
0  1  2  3  4  5  6  7  8  9  10    - sequence number
      I           I           I     - I if keyframe
```
Let's say we have the SPS and PPS (delivered out-of-band) parsed and payloaded into an RTP packet `"P"` that we may inject into the stream at will.
Let's also assume that we will drop packets until we get the first keyframe, then inject the parameters before every keyframe in the stream.

If we inject some packets into the stream, we have to change the sequence numbers of all of the following packets.
For now, let's say we're just going to assign them entirely new numbers, sequentially.

If we receive the packets in sequence, we'll be sending:
```
P  2  3  4  5  P  6  7  8  9  P  10    - original sequence numbers
0  1  2  3  4  5  6  7  8  9  10 11    - new sequence numbers
```

Suppose, however, that packet 6 arrived out of order, after packets 7 and 8:
```
0  1  2  3  4  5  7  8  6  9  10       - packets received by client
```
We're just checking if something is a keyframe, so this means we're sending
```
P  2  3  4  5  7  8  P  6  9  P  10
0  1  2  3  4  5  6  7  8  9  10 11    - new sequence numbers
```
The issue is obvious: the packet with the original sequence number 6 is now *after* the packet with the original sequence number 7.
This means that we can't simply number the packets sequentially in the order we receive them.

OK, then suppose `new = old + offset`, where `offset` - amount of packets injected by us up to that point.
In sequence:
```
P  2  3  4  5  P  6  7  8  9  P  10
2  3  4  5  6  7  8  9  10 11 12 13    - new sequence numbers
```
Seems good, right? It's certainly better, as it will correctly number regular frames, which arrived out of order.
Unfortunately, this won't be the case with keyframes arriving out of order. Consider, once again, the previous example:
```
P  2  3  4  5  7  8  P  6  9  P  10
2  3  4  5  6  8  9  ?? 7  10 11 12
```
We only have one spot left (number 7) for both parameters and I-frame 6. Not ideal.

Alright then, let's say `new = old * 2`, and if we need to inject a packet, just use the free number.
In sequence:
```
P  2  3  4  5  P  6  7  8  9  P  10
3  4  6  8  10 11 12 14 16 18 19 20
```
And, in the changed order:
```
P  2  3  4  5  7  8  P  6  9  P  10
3  4  6  8  10 14 16 11 12 18 19 20
```
Problem solved? Not quite. The issue is, this might cause some element downstream to think there's packet loss present,
because of the gaps. This, in turn, may cause RTCP NACKs to be sent for nonexistent packets, and possibly other unpleasant
things to follow.

Of course, there's the option of adding the parameters before EVERY frame - if you're alright with wasting a lot of bandwidth, that is.

Another possible solution would be to use an ordering buffer of an arbitrary size. This, however, introduces more delay...

And now the fun part: To make things simpler, we assumed an RTP packet to contain exactly one frame. This doesn't have to
be the case! RTP packets can also have multiple different frames (with the same or different timestamps), or just a part
of a single frame in their payload. Refer to [RFC 3984](https://www.rfc-editor.org/rfc/rfc3984) if you wish to learn more.

> All of this contains some oversimplifications (Access Units =/= Network Abstraction Layer Units). Refer to
> [the official H.264 specification](https://www.itu.int/rec/dologin_pub.asp?lang=e&id=T-REC-H.264-201602-S!!PDF-E&type=items)
> if you wish to learn even more.

### But what if we didn't have to add more packets?
Suppose we simply took each packet with an I-frame, extracted its payload, attached SPS and PPS before it,
and payloaded it again, onto a single RTP packet with the same ordering number, just maybe a different packet type
and a different size? No numbering/reordering issues then!

Unfortunately, that would be too good to be true. Increasing payload size means you run the risk
of exceeding the network's MTU, in which case, the modified packets will get dropped along the way.

But surely we could add parameters to payload, then fragment if necessary so that each individual packet is below the MTU?
Maybe that's the solution we've been looking for-- oh, wait, this runs headfirst into the very same numbering issues as before.

To summarise, in most cases, if one wishes to inject H264 parameters into the RTP stream itself, depayloading, parsing
and repackaging the entire stream is the only viable option.
