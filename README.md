# ADO Pipelines agent cross-contamination test helpers

## Problem

Because Azure DevOps _("ADO")_ [Managed DevOps Pool](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/agents#managed-devops-pools-agents) _("MDOP")_ agent VMs are not fully ephemeral, sequential workloads on the same VM may be exposed to residual state from prior runs.  Contamination vectors may include:

* **environment variables**:  set by a prior pipeline run and not cleaned up
* **filesystem artifacts**:  temp files, cached packages, build outputs, or leaked secrets written to disk
* **cached credentials**:  tool credential caches _(e.g. npm, NuGet, Docker)_ persisted into a VM operating system user profile
* **installed software**:  tools installed by a prior run that may alter behavior of subsequent runs

## Purpose

The Azure Pipelines YAML files in this repo is meant to help ADO implementers observe inter-run residual state "cruft."

* _(e.g. being able to see that the previous job's environment variable `name` is still set to `World`)_

This can help inform appropriate recommendation-making about mitigations such as:

* MDOP TTL tuning
* Allowed usability scope for each ADO agent pool _(if controllable centrally; may not be if ADO project admins can override settings?  TODO: research)_
* Pipeline code hygiene conventions _(e.g. explicit cleanup steps)_ in need of widespread training
* Monitoring and observability of what happens during pipeline runs _(if available; ADO might not surface a lot?)_

## Invocable YAML templates

| File | Purpose |
|---|---|
| [`010_leave_some_cruft.yml`](.azure_pipelines_yaml_files/templates/010_leave_some_cruft.yml) | Deliberately writes 8 types of cross-run artifacts to the agent |
| [`020_detect_if_cruft.yml`](.azure_pipelines_yaml_files/templates/020_detect_if_cruft.yml)  | Checks for the presence of those artifacts — run against the same pool as `010` if you'd like to see if you land on a "cross-contaminated" VM |
| [`030_cleanup_any_cruft.yml`](.azure_pipelines_yaml_files/templates/030_cleanup_any_cruft.yml) | Removes all artifacts written by `010` |

## Use

Reference these as step templates from a parent pipeline, such as the demos here or one of your own.

1. Run `010` on your pool.
2. Then trigger `020` repeatedly _(because if the pool size is >1, it might take a while to land on a VM "cross-contaminated" by `010`; hopefully you'll get there before MDOP TTL kills off the `010`-contaminated VM)_ until you see the same VM hostname appear in the agent identity step.
    * A `FOUND` result in the "detection" stage's steps demonstrates the potential **inter-run cross-contamination** via use of that pool.
3. Run `030` if you'd like to clean up the MDOP state afterward, so as to begin a fresh demo.

## Included YAML pipeline demos

1. [`demo_as_one_job.yml`](.azure_pipelines_yaml_files/demo_as_one_job.yml) should always produce a "WARNING" in the "detection" steps, because it runs all of the templated steps back-to-back as one big job.  If it doesn't do so, something went wrong with the code of the templates it invokes.
2. [`demo_as_separate_jobs.yml`](.azure_pipelines_yaml_files/demo_as_separate_jobs.yml) should:
    * always report "clean" in the "detection" steps if you're just running it with built-in straight-out-of-the-box [public Microsoft-hosted Azure Pipelines agent pools](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/agents#microsoft-hosted-agents) like `ubuntu-latest`, because those are always supposed to be ephemeral per job, and `demo_as_separate_jobs.yml` breaks up the template invocations into separate back-to-back jobs.
    * variably report "clean" vs. "WARNING" depending on what kind of "agent pool" you've got it running on.
        If it's one subject to cross-contamination, in fact, skipping the "cleanup" stage on a first run and skippng "leave" stage on a 2nd run will likely show cruft in _both_ runs' "detect" stages, as long as the gap between two runs doesn't exceed your MDOP TTL.  😬
        * Don't be afraid to try this yourself.  This is actually the point that you need to make to colleagues about MDOP TTL security vs. cost/performance tradeoffs, and this codebase is meant to help you prove your point.

## Author notes to self TODO

1. When testing `demo_as_separate_jobs.yml`, remember it's not very useful for demonstrating cross-contamination if the pool has more than 1 VM in it.  Don't wonder why it's not "failing" as expected if you work with a >1-size MDOP during demos.
2. Validate that `030` actually works by running `020` after it.  You still haven't had a chance to do this on a real MDOP just yet.

-Katie Kodes, 4/9/26