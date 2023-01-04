# Active Speaker Detection

A common feature in video conferencing software is active speaker detection. It is a small feature, but one that the user would feel weird without. Oftentimes, it is visualised by highlighting the user's tile while they are talking.
Other use cases include deciding which participants should be shown on the screen.
This can be implementedby using historical data about their voice activity.

As you can see, voice activity detection is a small, but important piece of any WebRTC implementation.
WebRTC standard provides tools that make it easy and convenient to implement this in your Selective Forwarding Unit.

## Audio Level Header extension
RFC 6464 defines an Audio Level Header extension, which is very useful in the context of Voice Activity Detection, as it takes the implementation load of the SFU.

The Extension carries the information about the audio level in `-dBov` with values from 0 to 127. Please pay attention to the minus, as it makes it so that the louder it gets, the lower the value becomes.
The fact that values are negative is a consequence of a definition of the `dBov` unit.

RFC 6464 also defines an optional flag bit "V". When the use of it is negotiated, it indicates whether the encoder believes the audio packet contains voice activity.

### What is `dBov`
`dBov` is defined as the level (in decibels) relative to the overload point of the system.
In this context, an overload point is the highest-intensity signal encodable by the payload format.

All of that is well and good, but how does that translate into something readable? Well, let's start with the "highest-intensity signal encodable by the payload format".
It's simply the loudest volume that you could encode into the given codec, usually OPUS.

Then there is an issue of decibels.
There are two things that you need to know about decibels:
1. It's a unit of relative intensity, meaning you need to define a base energy level.
In the case of the commonly used dB scale, that relative number is roughly the quietest sound that a human can hear.
In the case of `dBox`, this level is the loudest sound that can be encoded.
2. Decibel is a logarithmic value.
The base of the logarithm differs on application, but usually it's 10.
Simply put, the `db` value tells you how many times you need to multiply the base level by 10 to get the measured value.
In the case of negative numbers, we divide by 10 instead of multiplying.

## Audio Level processing
You could use the flag bit "V" if available, but we don't recommend using it for the production environment for 2 reasons:
1. It is optional, so you cannot be sure it will always be there.
2. Implementation isn't standardized, meaning that you can have inconsistent behavior depending on the sender.

It is therefore worth considering implementing your algorithm based purely on the `level` field.
Hopefully, you now have some understanding of the value in the `level` field, and we can jump right into the algorithm.
Don't worry, it's not difficult at all.

### Detecting speech
The basic idea behind speech detection is a *threshold*.
It is a volume that separates silence from speech.
Marking audio packets as either speech or silence is straightforward:
- if the volume is higher than the threshold, the packet should be considered speech
- if otherwise, it should be considered silence.

However, human speech doesn't have a constant volume.
Humans also take small breaks inbetween words.
At 10 Hz, you can detect these, so while taking into consideration that only one packet at a time isn't good enough, we need to look at the bigger picture.

The simplest and most effective way to implement it is to use a time window.
You then calculate the average volume of the packet and apply the threshold to that value.
If the threshold is exceeded, speech is marked on the audio track.

There are two parameters in this algorithm that you can tweak:
1. `threshold` to tweak the sensitivity concerning the volume. You should tweak it if you find that your implementation doesn't detect actual speech at all, meaning that it is too high or, on the contrary, it interprets background noise as speech.
2. Time window duration to tweak sensitivity to detect short, but loud sounds.

### Detecting silence
Detecting silence uses the same idea of a time window and comparing average volume to the threshold as detecting speech.
We have found it rather tricky to tweak the values in such a way that speech is detected swiftly, and silence detection isn't too aggressive.
For that reason, we need to apply an additional condition to silence detection -a time for which the average value must be below the threshold to mark voice activity on the audio stream as silence.

### Things to watch out for during implementation
In no particular order:
- WebRTC usually uses UDP under the hood, so packets will arrive out of order. You probably don't want to get a jitter buffer involved, so make sure that your time window implementation can handle out-of-order and possibly even late packets.
- Remember that you're dealing with `-dBov`. The absolute value for silence is `127`, and the loudest possible sound gets `0`!

