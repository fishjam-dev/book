# Active Speaker Detection

A common feature in video conference software is active speaker detection. It is a small feature but the one that the user would feel weird without. Often times, it takes the form of a gentle highlight of the user's tile when they are talking.
Another usecases include deciding which participants should be shown on the screen.
This can be implement eg. using historical data about their voice activity.

As you can see, voice activity detection is a small but important piece of any WebRTC implementation.
WebRTC standard provides tools that make it easy and convienient to implement in your Selective Forwarding Unit.

## Audio Level Header extension
RFC 6464 defines an Audio Level Header extension, very useful in the context of Voice Activity Detection, as it takes the implementation load of the SFU.

The Extension carries the information about the audio level in `-dBov` with values from 0 to 127. Please mind the minus, it makes it so that the louder it gets, the lower the value.
The fact that values are negative is a consequence of a definition of `dBov` unit.

RFC 6464 also defines an optional flag bit "V". When the use of it is negotiated, it indicates whether the encoder believes the audio packet contains voice activity.

### What is `dBov`
`dBov` is defined as the level, in decibels, relative to the overload point of the system.
In this context, an overload point is the highest-intensity signal encodable by the payload format.

All well and good, but how that translates into something readable? Well, let's start with "highest-intensity signal encodable by the payload format".
It's simply the loudest volume that you could possibly encode into the given codec, usually OPUS.

Then there is an issue of decibels.
**!!WARNING!! Physics intensifies**
There are two things that you absolutely need to know about decibels:
1. It's a unit of relative intensity, meaning that you need to define a base enegry level.
In case of commonly used dB scale that relative number is roughly thequitest sound that a human can hear.
In case of `dBox`, this level is the loudest sound that can be encoded.
2. Decibel is a logartmic value.
Base of the logarithm differs on application, but usually it's 10.
Simply put, `db` value tells you how many times you need to multiply the base level by 10 to get measaured value.
In case of negative numbers, we divide by 10 instead of multiplying.

## Audio Level processing
You could use the flag bit "V" if available, but we don't recommend using it for production environment, for 2 reasons:
1. It is optional, so you cannot be sure it will always be there.
2. Implementation isn't standarized, meaning that you can inconsistent behavior depending on the sender.

It is therefore worth considering implementing your own algorithm based purely on the `level` field.
Hopefully you know have some understanding of the value in the `level` field and we can jump right into the algorithm.
Don't worry, it's not difficult at all.

### Detecting speech
Basic idea behind speech detection is a *threshold*.
It is a volume that separates silence from speech.
Marking audio packets as either speech or silence is straight forward:
- if the volume is higher than the threshold, the packet should be considered speech
- otherwise, it should be consideded silence.

However, human speech doesn't have a constant volume.
Humans also take small breaks between words.
At 10 Hz, you can absolutely detect these and so, considering only one packet at a time just isn't good enough, we need to look at the bigger picture.

The simplest and effective way to implement it is to use a time window.
You then calculate the average volume of the packet and apply the threshold to that value.
If it exceeds the threshold, you mark speech on the given audio track.

There are two parameters in this algorithm that you can tweak:
1. `threshold` to tweak the sensitivity in regarrds to the volume. You should tweak it if you find that your implementation doesn't detect actuall speech at all, meaning that it is to high or, on the contrary, it interprets background noise as speech.
2. Time window duration to tweak sensitivity to short, but loud sounds.

### Detecting silence
Detecting silence uses the same idea of time window and comparing average volume to the threshold as detecting speech.
We have however found it rather tricky to tweak the values in such a way that the speech is detected swiftly and the silence detection isn't too agrresive.
For that reason, we need to apply an additional condition to silence detection.
A time for which the average value must be below threshold to mark voice activity on the audio stream as silence.

### Things to watch out for during implementation
In no particular order:
- WebRTC usually uses UDP under the hood, so packets can, and will, arrive out of order. You probably don't want to get a jitter buffer involved, so make sure that your time window implementation can handle out-of-order and possibly even late packets.
- Remeber that you're dealing with `-dBev`. An absolute value for silence is `127` and the loudest possible sound gets `0`!

## References
1. RFC 6464 "A Real-time Transport Protocol (RTP) Header Extension for Client-to-Mixer Audio Level Indication" - [RFC Editor](https://www.rfc-editor.org/rfc/rfc6464)
2. Definition of Decibel
