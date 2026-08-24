# aspect_tools_telemetry

> [!NOTE]
> This repository uses the [Aspect CLI](https://github.com/aspect-build/aspect-cli) for CI and local development.
> See the [docs](https://docs.aspect.build/cli/overview) and [install instructions](https://docs.aspect.build/cli/install) to get started.

Aspect's ruleset telemetry Bazel module.

> [!TIP]
> For the full story — what is collected and why, how the data is protected, and
> every way to opt out — see [our blog post](https://aspect.build/blog/bazel-ecosystem-usage-stats).
> The public statistics built from this data live at
> [aspect.build/open-source/stats](https://aspect.build/open-source/stats).

This package defines a Bazel extension which allows for rulesets to report usage to Aspect, allowing us to estimate the install base of Bazel, rulesets, and monitor trends in the ecosystem such as library usage and Bazel versions.

## When reporting occurs

`aspect_tools_telemetry` is implemented as a Bazel module which performs side-effects.
This means that telemetry is collected at repository granularity only when Bazel modules are invalidated and re-evaluated.

Examples:
- A user adding a new Bazel dependency will invalidate modules and trigger reporting
- A user making a local code change and performing a build will not trigger reporting

## The notice

The first time the module would report, it prints a notice and sends nothing; reporting
starts on a later module-graph evaluation, which leaves a window to opt out before
anything is sent. The record that the notice was shown persists via
`MODULE.bazel.lock` (Bazel's extension facts). A repository without a lockfile cannot
carry that record, so it gets a louder notice and reports on the same invocation; on the
ephemeral CI machines where this typically happens, the notice lands in every job's log,
so the opt-out is effectively per repository rather than per machine.

## Controlling reporting

The telemetry module honors `$DO_NOT_TRACK` and will disable itself if this variable is set.

The telemetry module can be controlled at a finer granularity with the `$ASPECT_TOOLS_TELEMETRY` environment variable.
`$ASPECT_TOOLS_TELEMETRY` is a comma joined list of reporting features using Bazel's set notation.

Some of the collector features can be overridden or salted for further privacy if so desired.

- `$ASPECT_TOOLS_TELEMETRY_SALT` is a value which will be included whenever computing a hash or ID.
  This allows you to salt correlation IDs if you so choose. For a public repository, set it as a
  CI secret rather than committing it, or the salt is as public as the file the ID derives from.
- `$ASPECT_TOOLS_TELEMETRY_ENDPOINT` overrides where reports are sent. Point it at your own
  collector and reports go there instead of to Aspect; the payload is the same `report.json`,
  so anything that accepts a JSON POST works.

### Example `.bazelrc` configurations

``` shell
# Disable entirely with the industry-standard variable (https://consoledonottrack.com)
common --repo_env=DO_NOT_TRACK=1

# Salt every hash so the values are unrecomputable without it; any value works,
# e.g. from `openssl rand -hex 8`. For a public repository, prefer setting it
# as a CI secret over committing it.
common --repo_env=ASPECT_TOOLS_TELEMETRY_SALT=b2d1a30326e6ba91

common --repo_env=ASPECT_TOOLS_TELEMETRY=all  # enabled (default)
common --repo_env=ASPECT_TOOLS_TELEMETRY=deps # only report aspect deps

common --repo_env=ASPECT_TOOLS_TELEMETRY=     # disabled
common --repo_env=ASPECT_TOOLS_TELEMETRY=-all # also disabled
common --repo_env=ASPECT_TOOLS_TELEMETRY=-id_day # just disable the day-scoped repo ID
```

## Reporting features

- `arch`: The arch per `repository_ctx.os.arch`
- `bazel_version`: The version of Bazel
- `bazelisk`: Whether the `bazelisk` tool is being used
- `ci`: Is the build occurring in CI/CD or locally
- `counter`: The build counter if available
- `deps`: The modules in `MODULE.bazel.lock` that were resolved from a registry, with versions
- `has_bazel_prelude`: Does the project use a `prelude_bazel`
- `has_bazel_tool`: Does the project use a `tools/bazel` script
- `has_bazel_workspace`: Does the project still have a `WORKSPACE` file
- `id_day`: A day-scoped hash of the repo, allowing same-day report deduplication; not linkable across days
- `os`: The os per `repository_ctx.os.name`
- `runner`: The CI/CD system being used if any

No user or organization identifiers are collected. The stable repository
ID that feeds `id_day` is computed on-device and never leaves the machine.

## Example exploration

The included examples/simple submodule provides a sandbox for easily testing the telemetry module's behavior.

``` shellsession
❯ cd examples/simple

# Default unconfigured behavior
❯ bazel build \
    --repo_env=CI=1 \
    --repo_env=DRONE_BUILD_NUMBER=678 \
    --repo_env=GIT_URL=http://github.com/aspect-build/tools_telemetry.git \
    //:report.json && cat bazel-bin/report.json
INFO: Analyzed target //:report.json (7 packages loaded, 10 targets configured).
INFO: Found 1 target...
Target //:report.json up-to-date:
  bazel-bin/report.json
INFO: Elapsed time: 0.300s, Critical Path: 0.03s
INFO: 2 processes: 1 internal, 1 darwin-sandbox.
INFO: Build completed successfully, 2 total actions
{
 "tools_telemetry": {
   "arch": "aarch64",
   "bazel_version": "8.3.1",
   "bazelisk": true,
   "ci": true,
   "counter": "678",
   "deps": {
      "aspect_tools_telemetry": "0.0.0",
      "simple-example": "0.0.0"
   },
   "has_bazel_prelude": false,
   "has_bazel_tool": false,
   "has_bazel_workspace": false,
   "id_day": "1c065d5f9c01ac06ba25ff2fb6f7db6c38d29cf3",
   "os": "mac os x",
   "runner": "drone"
}%

# Disabled behavior
❯ bazel build \
    --repo_env=CI=1 \
    --repo_env=BUILD_NUMBER=678 \
    --repo_env=JENKINS_HOME=$HOME \
    --repo_env=GIT_URL=http://github.com/aspect-build/tools_telemetry.git \
    --repo_env=DO_NOT_TRACK=1 //:report.json \
    && cat bazel-bin/report.json
INFO: Analyzed target //:report.json (7 packages loaded, 10 targets configured).
INFO: Found 1 target...
Target //:report.json up-to-date:
  bazel-bin/report.json
INFO: Elapsed time: 0.071s, Critical Path: 0.00s
INFO: 1 process: 1 action cache hit, 1 internal.
INFO: Build completed successfully, 1 total action
{}%
```

## Report inspection

For transparency reports are persisted into the Bazel configuration and can be inspected as `@aspect_tools_telemetry_report//:report.json`.

``` shellsession
❯ cat $(bazel info output_base)/external/*aspect_tools_telemetry_report/report.json
```

## What happens to reports

Reports are received by infrastructure Aspect operates, and are handled as follows:

- `id_day` is re-keyed at ingestion with a server-side secret that rotates daily; each
  day's outgoing secret is destroyed, so a closed day's stored IDs cannot be matched
  back to any repository.
- Fields collected by older versions of this module but not current ones (`user`,
  `org`, `id`, `shell`, `has_bazel_module`) are discarded at ingestion and never stored.
- Raw reports are deleted after at most 365 days, and reports that fail processing
  after 30 days.
- The aggregate statistics built from reports are published for the community at
  https://aspect.build/open-source/stats, with the dataset behind the charts at
  https://stats.aspect.build/stats.json.

## Privacy policy

Data collected by this telemetry package is reported to Aspect Build Systems Inc. and governed under our privacy policy.

For more please see https://www.aspect.build/privacy-policy
