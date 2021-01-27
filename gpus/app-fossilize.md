Fossilize
=========

## Captures and Replays

- capture with apis
  - `Fossilize::create_database` to create/open a `.foz` database
    - it returns a `Fossilize::DatabaseInterface` to manipulate the database
  - `StateRecorder recorder; recorder.init_recording_thread(db)` to initialize
    the recorder
  - from this point on, call `record_*` such as `record_application_info` or
    `record_sampler` to record objects to the db
  - when done, destroy the record to flush and stop the recording thread
- replay with apis
  - define app-specific `ReplayerInterface`
  - `StateReplayer replayer; replayer.parse(iface, db, ...)` to replay the
    database
- capture with `VK_LAYER_fossilize`
  - automatic create a db and invoke `StateRecorder`
  - `FOSSILIZE_DUMP_PATH_ENV`

## Database

- `fossilize-convert-db` unpacks a `.foz` database to (many) json files
- it looks like each `StateRecorder::record_*` call records a `Vk*Info` to a
  json snippet, and is saved to the database

## `fossilize-replay`
