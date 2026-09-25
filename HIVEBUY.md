# Hivebuy fork

Fork of [danihodovic/celery-exporter](https://github.com/danihodovic/celery-exporter), tracked in
Linear as HI-13110.

## Why a fork

The last image upstream published to Docker Hub (`0.12.2`) is from July 2025. Upstream `master`
has since gained things we need (`celery_task_queue_wait_time`, the `exception` label on
`celery_task_failed_total`, purging of offline workers) but has not cut a release. We also carry
three small patches that are not upstream yet.

## Branches

- `master`: mirror of upstream `master`. Never commit here.
- `hivebuy`: upstream `master` plus our patches and this file. This is what we build and run.

## Patches on `hivebuy`

1. `celery_task_retried_total` carries an `exception` label. The `task-retried` event already
   has the exception that caused the retry; upstream drops it. A retry without an exception is
   labelled `UnknownException`.
2. `celery_queue_length` on Redis sums the priority shards. Kombu keeps one list per priority
   step (`<queue>`, `<queue>:3`, `<queue>:6` with our `sep=:`). Upstream reads only the bare
   name, which is empty on our setup because tasks are published at priority 3 and 6.
3. `--defer-new-series SECONDS` (default 0, off). A series created by an event (a task
   failing with an exception not seen before, a fast task on a worker that is new) is
   otherwise first exported with a value of 1, and `increase()` / `rate()` never count that
   first event. With a delay set, the new series is created at zero and its first updates are
   applied on a scrape after the delay, so the scraper sees 0 first. Set it above the scrape
   interval (ours is 60 s, we run 120).

All three are candidates for upstream pull requests. Drop a patch here once upstream has it.

## Image

`.github/workflows/hivebuy-image.yml` tests and publishes every push to `hivebuy`:

```
ghcr.io/hivebuy/celery-exporter:hb-<short sha>
ghcr.io/hivebuy/celery-exporter:hb-latest
```

Infrastructure pins the `hb-<short sha>` tag (`modules/celery-exporter` in
`hivebuy/infrastructure`). Never deploy `hb-latest`.

## Updating from upstream

```
git fetch upstream
git switch master && git merge --ff-only upstream/master && git push origin master
git switch hivebuy && git rebase master && git push --force-with-lease origin hivebuy
```

Then bump the pinned tag in infrastructure, dev first.
