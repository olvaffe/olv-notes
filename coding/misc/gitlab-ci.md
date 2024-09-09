GitLab CI
=========

## CI/CD YAML syntax reference

- <https://docs.gitlab.com/ee/ci/yaml/>
- Global Keywords
  - `default` Custom default values for job keywords.
    - each default keyword is copied to every job that does not already have
      it defined
      - each job can customize what to copy with `inherit`
  - `include` Import configuration from other YAML files.
  - `stages` The names and order of the pipeline stages.
    - ordered list of stages, defaulting to `.pre`, `build`, `test`, `deploy`,
      and `.post`
    - jobs in the same stage run in parallel
    - jobs in the next stage run after jobs from the previous stage complete
      successfully
  - `variables` Define CI/CD variables for all job in the pipeline.
  - `workflow` Control what types of pipeline run.
- Header Keywords
  - `spec` Define specifications for external configuration files.
- Job Keywords
  - `after_script` Override a set of commands that are executed after job.
  - `allow_failure` Allow job to fail. A failed job does not cause the pipeline to fail.
  - `artifacts` List of files and directories to attach to a job on success.
    - artifacts are available for downloads in the ui
    - artifacts are automatically copied to jobs in later stages,
      - each job can customize what to copy with `dependencies`
  - `before_script` Override a set of commands that are executed before job.
  - `cache` List of files that should be cached between subsequent runs.
  - `coverage` Code coverage settings for a given job.
  - `dast_configuration` Use configuration from DAST profiles on a job level.
  - `dependencies` Restrict which artifacts are passed to a specific job by providing a list of jobs to fetch artifacts from.
    - default to all jobs in earlier stages
  - `environment` Name of an environment to which the job deploys.
  - `extends` Configuration entries that this job inherits from.
  - `identity` Authenticate with third party services using identity federation.
  - `image` Use Docker images.
  - `inherit` Select which global defaults all jobs inherit.
  - `interruptible` Defines if a job can be canceled when made redundant by a newer run.
  - `manual_confirmation` Define a custom confirmation message for a manual job.
  - `needs` Execute jobs earlier than the stage ordering.
  - `pages` Upload the result of a job to use with GitLab Pages.
  - `parallel` How many instances of a job should be run in parallel.
  - `release` Instructs the runner to generate a release object.
  - `resource_group` Limit job concurrency.
  - `retry` When and how many times a job can be auto-retried in case of a failure.
  - `rules` List of conditions to evaluate and determine selected attributes of a job, and whether or not it’s created.
  - `script` Shell script that is executed by a runner.
  - `secrets` The CI/CD secrets the job needs.
  - `services` Use Docker services images.
  - `stage` Defines a job stage.
    - default to `test`
  - `tags` List of tags that are used to select a runner.
  - `timeout` Define a custom job-level timeout that takes precedence over the project-wide setting.
  - `trigger` Defines a downstream pipeline trigger.
  - `variables` Define job variables on a job level.
  - `when` When to run job.
- deprecated
  - `image`, `services`, `cache`, `before_script`, and `after_scrip` as global
    keywords are deprecated
  - `only` and `except` are deprecated

## CI/CD pipelines

- <https://docs.gitlab.com/ee/ci/pipelines/>
- pipelines comprise jobs and stages
  - jobs are executed by runners
  - jobs in the same stages are executed in parallel
  - if all jobs in a stage succeed, the pipeline moves on to the next stage
- types of pipelines
  - branch pipelines
    - they run when new commits are pushed to branches
    - they run by default, no configuration is needed
  - merge request pipelines
    - they run when a MR is created or the source branch of a MR is changed
    - they run on the source branch only, ignoring the target branch
    - they do not run by default and need to be configured
      - `rules:if: $CI_PIPELINE_SOURCE == 'merge_request_event'`
  - merge result pipelines
    - they are like merge request pipelines, except they run on the merged
      result of the source branch and the target branch
    - they do not run by default and need to be configured
      - `rules:if` like for merge request pipelines
      - additionally, `Settings > Merge requests > Enable merged results pipelines`
  - merge trains
    - they are like merge result pipelines, except when there are already MRs
      pending merging and under testing by their respective merge result
      pipelines, the new merge result pipeline run on the merged result of the
      source branch, the target branch, and all pending MRs
    - they do not run by default and need to be configured
      - see merge result pipelines
      - additionally, `Settings > Merge requests > Enable merged trains`
