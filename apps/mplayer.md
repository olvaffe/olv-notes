FFMPEG and MPLAYER
==================

## codecs.conf

- look at an entry for example
- videocodec theora
    info "Theora (free, reworked VP3)"
    status working
    fourcc theo,Thra
    driver theora
    dll libtheora
    out YV12
- mplayer -vc help gives:
  theora      theora    working   Theora (free, reworked VP3)  [libtheora]
  |videocodec |driver   |status   |info                        |dll
  where drivers are vd, dll are vd specific subformat
- mplayer -vfm help gives a list of vd:
  decoders from libmpcodecs/vd_*.c
- mplayer -vf help gives a list of vf:
  filters from libmpcodecs/vf_*.c
- mplayer -demuxer help gives a list of demuxers:
  demuxers from libmpdemux/demux_*.c

## libavformat

- config.mak decides which files to compile
  config.h helps decide what goes into allformats.c
- mplayer -lavfdopts format=help <somefile> to list supported formats

## libavcodec

- config.mak decides which files to compile
  config.h helps decide what goes into allcodecs.c

MPContext
- has a stream_t *stream for stream transport
- has a demuxer_t *demuxer to demux the stream into
  demux_stream_t *d_audio, *d_video, *d_sub
- has a sh_audio_t *sh_audio for audio stream header
- has a sh_video_t *sh_video for video stream header
- has a mixer_t mixer to interface hw mixer
- both sh_audio_t and sh_video_t have filter chains

libmpcodecs/vd.c
- mpcodecs_config_vo: config vo
- at the end of filter chain is ([pp])->[vo]
- vo filter is a wrapper to a vo driver (x11, xv, etc.)
- for each format (sh->codec->outfmt) the codec can output, 
  the filter chain is asked for its capabilities.  If it has hw soft support, done.  
- if no support at all, [scale] is prepended to form
  [scale] -> ([pp]) -> [vo]
- and the process restarts
- [scale] filter is a wrapper to libswscale for scaling and conversion


android mplayer_service
- mplayer owns a stream and pause looping for transaction
- when a surface is given (start), output to the surface from beginning
- transaction can pause/unpause output and remove surface (stop)
- START_TRANSACTION(surface, filename)
- STOP_TRANSACTION()
- PAUSE_TRANSACTION()
- UNPAUSE_TRANSACTION()
- transaction id starting from 1
- -idle and -slave: player_idle_mode and slave_mode
- mp_input_add_key_fd/mp_input_add_cmd_fd/mp_input_add_event_fd
- impl
  mplayer runs in idle mode, waiting for file to be given (before vo)
  binder.c calls mp_input_add_event_fd to interface /dev/binder
  When START_TRANSACTION is received, return MP_CMD_LOADFILE and surface is set to vo_android
  vo_android preinit with success and starts playing
  When PAUSE/UNPAUSE, return MP_CMD_PAUSE
  When STOP, return MP_CMD_STOP
  vo_android uninit is called

mplayer.c:main
- beginning: parse configs, init input
- play_next_file: decide stream, open subs, open streams
- goto_enable_cache: caching, open demuxer, demux and decides a/v streams
- MAIN: playing and looping
- goto_next_file: decide next file and goto play_next_file or end mplayer
  
MAIN
- SETUP AUDIO:
- START PLAYING: entering main loop
- PLAY AUDIO: fill_audio_out_buffers
- PLAY VIDEO: update_video to decode frames and buffer them.  mpctx->video_out->check_events()
- sleep_until_update: wait for next frame.  for 1 frame/second clip, it might sleep up to 1 second!
- FLIP PAGE: mpctx->video_out->flip_page()
- A-V TIMESTAMP CORRECTION: adjust_sync_and_print_status
- Auto QUALITY
- Handle PAUSE: pause_loop()
- Keyboard events, SEEKing 

PAUSE
- if OSD_PAUSE, enter pause_loop
- if any cmd without pausing == 4 (pausing_keep_force), unpause and OSD_PAUSE
? |FRAME1|FRAME2|FRAME3|
    1    2      3
? at 1, 'p' pressed
? at 2, osd_function is checked, THEN pause cmd is run and osd_function set to OSD_PAUSE
? at 3, osd_function is checked again and enter pause_loop

reinit_video_chain
- is called just before MAIN
- calls init_best_video_out to set up mpctx->video_out

idle mode
- before exiting mplayer, goto play_next_file again unconditionally
- in play_next_file section, loop waiting until a filename is decided

slave mode
- 

## ffmpeg encoding

- `libavformat/output-example.c`
- one buffer is allocated for AVFrame, another buffer is allocated for
  compressed data
- `avcodec_encode_video` encodes a frame and outputs to the second buffer.
