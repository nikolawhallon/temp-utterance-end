# temp-utterance-end

## Endpointing and its limitations

Deepgram's Endpointing feature uses an audio-based Voice Activity Detector (VAD) to determine when there is signal audio (e.g. speech) about background audio (e.g. noise).
Assuming the signal audio is always speech, this means it determines for each frame of audio (where the definition of "frame" is internal) whether there is speech present
or not. When the state of the audio goes from speech to a configurable duration of no-speech (by default 10 ms), Deepgram will chunk the audio at that point
and return results with the `speech_final` flag set to `true`. The duration of no-speech (following speech) required to trigger such an event is configurable
via the API parameter `endpointing`, for example `endpointing=1000` will require that 1000 ms of no-speech (following speech) was present.

This feature has its limitations - non-speech signal audio can interrupt it, for example. Suppose you set `endpointing=1000` and you say "Hello." followed
by 2000 ms of silence - chances are you will get a Deepgram transcription result with the transcript "Hello." and `speech_final` set to `true` - great.
However, suppose that 500 ms after you say "Hello." someone knocks on your door, or your phone starts to ring
(or a particularly loud car drives past the house) - these are all signal audio (i.e. not part of the background noise profile) but do not represent speech.
This is where an audio-based VAD could make mistakes - it may think that speech has started again, and therefore not recognize 1000 ms of silence after you
said "Hello." and Deepgram would then not return a result with `speech_final` set to `true` - indeed, since Deepgram streams results by default every
3-5 seconds in the absence of special events (like a `speech_final`/Endpointing message), Deepgram will probably return a finalized result regardless.

## UtteranceEnd and its limitations

Deepgram offers an alternative to the Endpointing feature, known as the UtteranceEnd feature. This feature looks at the word timings of both
finalized and interim results to determine if a sufficiently long gap in words has occured, and if it has it will return a json message of the
following shape:

```
{"type":"UtteranceEnd"}
```

where the duration of the "sufficiently long gap" is configurable via the API parameter `utterance_end_ms`. For example, if you set
`utterance_end_ms=1000` Deepgram will look for 1000 ms gaps between transcribed words and send an UtteranceEnd message if it finds one.
Since this relies on words and word timings, and not explicitly on the audio, it is immune to non-speech audio - no amount of door knocking
or phone ringing will get in the way.

However, because words are only available after the Deepgram model inferences audio, the resolution
of this approach is limited by the speed of inferenced results. By default, finalized Deepgram results are generated every 3-5 seconds,
and interim results are generated every 1 second, so trying to use a value for `utterance_end_ms` less than 1 second will not offer any
benefits over setting it to 1 second. This is also the reason why the API will return an error if you specify a value for `utterance_end_ms` but
do not specify `interim_results=true` (while technically the algorithm could still function, the experience would be terrible as it would
only have a resolution of 3-5 seconds!).

## Using both

The Endpointing and UtteranceEnd features function completely independently of each other, so it is definitely possible to use both at the same time.
In this case, you may want to trigger your app logic anytime you get a `speech_final` (which may be followed by an UtteranceEnd message which you
could then ignore), or if you receive an UtteranceEnd message with no preceeding `speech_final` message, you could trigger on that.

## Ultimate limitations

Ultimately, any approach to determine when someone has finished speaking is a heuristic one and can fail. Why? Because humans can resume talking
at any time for any reason (indeed some don't know when to stop!). Because of this, you might consider introduced special logic to your app
handling the cases where you think you have received an actionable transcript, you send it down your pipeline, and then new speech starts
being received by your app - you then cancel whatever tasks you may have triggered in your pipeline and start over, given the new input.
