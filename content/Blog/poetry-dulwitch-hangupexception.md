---
title: Poetry Install Dulwitch HangupException
date: 2023-12-08T09:33:24+01:00
publish: true
tags:
  - python
---

## The error

```
  HangupException

  The remote server unexpectedly closed the connection.

  at /usr/local/lib/python3.10/site-packages/dulwich/protocol.py:215 in read_pkt_line
      211│ 
      212│         try:
      213│             sizestr = read(4)
      214│             if not sizestr:
    → 215│                 raise HangupException
      216│             size = int(sizestr, 16)
      217│             if size == 0 or size == 1:  # flush-pkt or delim-pkt
      218│                 if self.report_activity:
      219│                     self.report_activity(4, "read")

The following error occurred when trying to handle this error:


  HangupException

  Host key verification failed.

  at /usr/local/lib/python3.10/site-packages/dulwich/client.py:1154 in fetch_pack
      1150│         with proto:
      1151│             try:
      1152│                 refs, server_capabilities = read_pkt_refs(proto.read_pkt_seq())
      1153│             except HangupException as exc:
    → 1154│                 raise _remote_error_from_stderr(stderr) from exc
      1155│             (
      1156│                 negotiated_capabilities,
      1157│                 symrefs,
      1158│                 agent,

Cannot install marker.

------
Dockerfile:32
--------------------
  30 |     COPY poetry.lock pyproject.toml ./
  31 |     
  32 | >>> RUN --mount=type=cache,target=$POETRY_CACHE_DIR poetry install --without dev --no-root
```

## Where I encountered the error

I was trying to install packages with poetry in a Dockerfile.

## How I traced it

There was no mention of `dulwitch` in `poetry.lock` or `pyproject.toml`. So it must come from poetry itself.

- I bumped poetry from 1.7.0 to 1.7.1 -> no change

Then I noticed this post on stack: https://stackoverflow.com/questions/74523531/importing-github-repo-as-dependency-in-poetry-in-docker-container

I checked my `poetry.toml` and found this:

```toml
marker = {git = "git@github.com:VikParuchuri/marker.git"}
```

I was installing one of the dependencies from source using git. I changed it to https:

```toml
marker = {git = "https://github.com/VikParuchuri/marker.git"}
```

And the error went away ;)

More on that here: https://github.com/python-poetry/poetry/issues/6999
