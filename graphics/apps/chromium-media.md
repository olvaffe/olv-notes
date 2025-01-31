Chromium Media
==============

## Media

- <https://chromium.googlesource.com/chromium/src.git/+/refs/heads/main/media/README.md#playback>
  - `<video>` and `<audio>`
    - each `blink::HTMLMediaElement` owns a `blink::WebMediaPlayerImpl`, with
      call stack
      - `blink::WebMediaPlayerImpl`
      - `content::MediaFactory::CreateMediaPlayer`
      - `content::RenderFrameImpl::CreateMediaPlayer`
      - `blink::HTMLMediaElement`
  - `blink::WebMediaPlayerImpl`
    - owns a `media::PipelineController` to coordinate
      - `media::DataSource`
      - `media::Demuxer`
      - `media::Renderer`
    - calls `DemuxerManager::CreateDemuxer` to create a `media::Demuxer`
      - `media::FFmpegDemuxer` demuxes the video file into
        `media::DemuxerStream`s
    - `CreateRenderer` is called when the pipeline needs a renderer
      - calls `media::RendererFactorySelector::GetCurrentFactory` to get the
        current factory
      - usually reaches `media::RendererImplFactory::CreateRenderer` to
        create a `media::RendererImpl`
        - `AudioRendererImpl`,
        - `VideoRendererImpl`
        - `GpuMemoryBufferVideoFramePool`
  - `media::RendererImpl`
    - owns a `media::AudioRenderer` and a `media::VideoRenderer` to decode
      `media::DecoderStream`s
- from `--v=3` log,
  - `DecoderSelector<>::SelectDecoderInternal` tries all decoders one by one
  - once a decoder is selected, `DecoderStream<>::OnDecoderSelected` is called
  - when an encoded buffer is ready,
    `DecoderStream<StreamType>::OnBufferReady` calls
    `DecoderStream<StreamType>::Decode`
- vaapi
  - it seems to be enabled by default on x11
    - `use_vaapi_x11`
    - `use_vaapi`
    - `VaapiVideoDecoder` feature is enabled by default and
      `VaapiVideoDecodeLinuxGL` feature is disabled by default
  - cmdline
    - `--ignore-gpu-blocklist`
- video decode trace view
  - renderer process
    - `MediaSource::StartAttachingToMediaElement`
    - `ChunkDemuxer::Initialize`
    - `DecoderSelector::SelectDecoder`
    - `VideoRendererImpl::Initialize`
    - `VideoDecoderStream::Read`
    - `VideoDecoderStream::ReadFromDemuxerStream`
    - `MojoVideoDecoder::Decode`
    - `MojoDecoderBufferWriter::Write`
    - `MojoDecoderBufferReader::Read`
    - `VideoDecoderStream::Decode`
  - gpu process
    - `MojoDecoderBufferReader::Read`
    - `MojoDecoderBufferWriter::Write`
    - `MojoVideoDecoderService::Decode`
    - `VideoDecoderPipeline::Decode`
    - `VideoDecoderPipeline::DecodeTask`
  - utility process
    - it seems to be doing OOP decoding where the real decoding happens on the
      utility process
    - `VideoDecoderPipeline::DecodeTask`
    - `VaapiVideoDecoder::HandleDecodeTask`
    - `VaapiVideoDecoder::Decode`
    - `VP9VaapiVideoDecoderDelegate::SubmitDecode`
    - `VaapiWrapper::MapAndCopyAndExecute`
  - summary
    - renderer `VideoDecoderStream::Read` returns a decoded frame
      - internally, it calls `VideoDecoderStream::Decode` to decode a frame
        which calls `MojoVideoDecoder::Decode`
      - `MojoDecoderBufferWriter::WriteDecoderBuffer` passes the encoded
        buffer over mojo
      - `ready_outputs_` holds the decoded frames
        - the type is `DecoderStream::Output` which is `VideoFrame`
        - it is updated by `DecoderStream<StreamType>::OnDecodeOutputReady`
    - gpu `MojoVideoDecoderService::Decode`
      - `MojoDecoderBufferReader::ReadDecoderBuffer` reads the buffer and
        invokes `VideoDecoderPipeline::Decode`
    - with oop decoding, gpu process offloads decoding to the utility process
- `VideoDecoderPipeline::Decode`
  - it calls `VideoDecoderPipeline::DecodeTask` on the runner thread which
    calls `decoder_->Decode`
  - `decoder_` is created by `VaapiVideoDecoder::Create`
  - when the frame is decoded/processed/converted, it calls
    - `VideoDecoderPipeline::OnFrameDecoded`
    - `VideoDecoderPipeline::OnFrameProcessed`
    - `VideoDecoderPipeline::OnFrameConverted`
      - this calls `client_output_cb_` to pass the decoded `VideoFrame` to the
        client
      - when the client is `MojoVideoDecoderService`,
        `MojoVideoDecoderService::OnDecoderOutput`
- `VaapiVideoDecoder::Decode`
  - it also has a `decoder_` that is created by `VaapiVideoDecoder::CreateAcceleratedVideoDecoder`
    - for vp9, it creates a `VP9VaapiVideoDecoderDelegate` and wraps the
      delegate in a `VP9Decoder`
  - it calls `VP9Decoder::Decode` which calls
    `VP9Decoder::DecodeAndOutputPicture` which calls
    `VP9VaapiVideoDecoderDelegate::CreateVP9Picture` and
    `VP9VaapiVideoDecoderDelegate::SubmitDecode`
  - `VP9VaapiVideoDecoderDelegate::CreateVP9Picture`
    - `vaapi_dec_->CreateSurface` is `VaapiVideoDecoder::CreateSurface`
      - it creates a `VideoFrame` using
        `PlatformVideoFramePool::GetFrame`
      - it imports the `VideoFrame` to a `VASurface` using
        `CreateNativePixmapDmaBuf` and
        `VaapiWrapper::CreateVASurfaceForPixmap`
  - `VP9VaapiVideoDecoderDelegate::SubmitDecode`
    - `VaapiWrapper::CreateVABuffer` create `VABuffer`s
    - `VaapiWrapper::MapAndCopyAndExecute`
      - uploads encoded data to `VABuffer`s
      - `vaBeginPicture`
      - `vaRenderPicture`
      - `vaEndPicture`

