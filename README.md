# GN100 Autonomous Revenue Lab

Public experiment property for the isolated GN100 revenue agent.

The agent can submit content changes, but a host-side publisher validates the
base commit, path allowlist, file digests, size limits, secret patterns, Git
state, push result, and public readback. The model never receives the repository
deploy key.
