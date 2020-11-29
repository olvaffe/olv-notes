Android Audio
=============

## `frameworks/base/media/`

* Music App has a service, `MediaPlaybackService`, for background playing
  * The real play core is `android.media.MediaPlayer`.
* The Java `android.media.MediaPlayer` is a wrapper to C++ `MediaPlayer`, which
  is provided by `libmedia/`.
  * It inherits `BnMediaPlayerClient` and is passed to `IMediaPlayerService` to
    create a `IMediaPlayer`.
* `mediaserver/` provides the executable `mediaserver`
  * It instantiates `AudioFlinger`, `MediaPlayerService`, and `CameraService`.
  * Their names are `media.audio_flinger`, `media.player`, and `media.camera`.
  * They are provided by `libs/audioflinger/`, `libmediaplayerservice/`, and
    `camera/libcameraservice/`.
  * In simulator setup, these services are provided by system server.
* When a file is provided to `MediaPlayer`, `setDataSource` is called
  * serice manager is asked for `media.player`, which has `IMediaPlayerService`.
  * `IMediaPlayerService->create` is called to create a `IMediaPlayer` for the
    sourece
  * `IMediaPlayer->start` is called to start playing.
* On the `mediaserver` side,
  * When it is asked to create a `IMediaPlayer` for a file, it inspects the file
    and one of `PVPlayer`, `MidPlayer`, or `VorbisPlayer` is created.  The
    player creates a thread waiting to handle the file.
  * A `MediaPlayerService::AudioOutput` is created as the sink to the player.
  * When start playing, the player's render thread is signaled.  It asks its
    sink, an `AudioOutput`, to open, which creates a `AudioTrack`.  PCM data are
    written to the sink.
  * The `AudioTrack` is just created is a wrapper to `IAudioTrack`, as returned
    by `audioFlinger->createTrack`.
* The services provided by `mediaserver` are for multimedia playback.  System
  server provides `AudioService` which is at a higher level and provides means
  to configure audioflinger.
  * It can adjust volumes or re-route the audio pathes.
  * It uses `AudioSystem` to access audioflinger.
  * Applications uses `AudioManager` to access this service.

## AudioFlinger

* It is the center of audio handling
* Have a look at its `Android.mk` for hw support
  * `libaudiointerface` might be used directly when `BOARD_USES_GENERIC_AUDIO`
    * it opens `/dev/eac`, which exists on qemu goldfish
  * It might also be linked to by `libaudio` from HAL.
    * There is a `libaudio` for every platform.
    * There is a `libaudio` which is a bridge to ALSA.
* In the constructor,
  * `AudioHardwareInterface` and `A2dpAudioInterface` are created
  * Two `AudioStreamOut` are created from them, one for each.
  * Two `MixerThread` are created for the `AudioStreamOut`s.
  * A `MixerThread::OutputTrack` of a2dp is created and is set as the output
    track of `mHardwareMixerThread`.
* In the `threadLoop` of `MixerThread`,
  * There is a `AudioMixer` to mix tracks
  * The mixed buffer is written to `mOutputTrack` or `mOutput`.  The former is
    to redirect audio data to a2dp.
