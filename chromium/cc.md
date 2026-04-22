Chromium cc
===========

## Frame Metrics

- `LayerTreeHostImpl`
  - it has
    - `CompositorFrameReportingController`
    - `FrameSequenceTrackerCollection`
    - `TotalFrameCounter`
    - `DroppedFrameCounter`
    - `BeginFrameTracker`
  - on events, it calls
    - `FrameSequenceTrackerCollection::StartSequence`
    - `FrameSequenceTrackerCollection::NotifyBeginImplFrame`
    - `FrameSequenceTrackerCollection::NotifyFrameEnd`
    - `FrameSequenceTrackerCollection::StopSequence`
- collection
  - `CompositorFrameReportingController`
    - `WillBeginImplFrame` creates a `CompositorFrameReporter`
      - this is called from `Scheduler::BeginImplFrame`
    - `AdvanceReporterStage` will advance the reporter to next stages until
      `DidSubmitCompositorFrame` destroys the reporter
    - `AddSortedFrame`
      - `FrameSequenceTrackerCollection::AddSortedFrame`
      - `FrameSequenceTracker::AddSortedFrame`
      - `FrameSequenceMetrics::AddSortedFrame`
  - `CompositorFrameReporter`
    - dtor calls `DroppedFrameCounter::OnEndFrame`
  - `DroppedFrameCounter`
    - it has a `FrameSorter` to batch and sort frames
    - `OnBeginFrame` adds `viz::BeginFrameArgs` to the sorter
    - `OnEndFrame` adds `FrameInfo` to the sorter
      - this causes the sorter to call `NotifyFrameResult` which calls into
        `CompositorFrameReportingController::AddSortedFrame`
- report
  - `FrameSequenceTrackerCollection::StopSequence`
    - `FrameSequenceMetrics::ReportMetrics`
  - `FrameSequenceTrackerCollection`
    - `StartSequence` creates a `FrameSequenceTracker` of the specified type
    - `StopSequence` destroys the tracker and calls
      `FrameSequenceMetrics::ReportMetrics`
  - `FrameSequenceTracker`
    - it has
      - `FrameSequenceMetrics`
  - `FrameSequenceMetrics`
    - `ReportMetrics` reports the metrics as uma histograms
- example: dropped frame
  - typically, while animating, we should see this sequence of events
    repeatedly
    - vsync -> renderer begin frame -> renderer composite frame -> aggregate frame -> draw -> swap
    - or, in trace view,
      - `Display::FrameDisplayed`
      - `DelayBasedBeginFrameSource::OnTimerTick`
      - `ExternalBeginFrameSource::OnBeginFrame`
      - `SingleThreadProxy::DoComposite`
      - `DisplayScheduler::OnBeginFrameDeadline`
      - `Display::DrawAndSwap`
  - but when begin frame and composite frame take too long inside blink and
    miss the vsync interval
    - viz does not call `OnBeginFrameDeadline` because there is no frame to
      composite
    - viz still schedules a begin frame on next vsync, but that frame is
      dropped by `Scheduler::FinishImplFrame` with
      `FrameSkippedReason::kDrawThrottled`
      - `DroppedFrameCounter::NotifyFrameResult` will emit a
        `DroppedFrameDuration` trace event
