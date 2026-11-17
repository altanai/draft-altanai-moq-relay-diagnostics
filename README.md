# Client-Visible Relay Diagnostics for MOQT

This repository contains the working source for the individual Internet-Draft
**Client-Visible Relay Diagnostics for Media over QUIC Transport (MOQT)**.

- [Current draft source](draft-altanai-moq-relay-diagnostics.md)
- [moq-transport issue #1692](https://github.com/moq-wg/moq-transport/issues/1692)

A proposal preview and working notes are maintained in the local IETF workspace.

The draft starts with the problem statement and requirements, then compares
three possible approaches: an explicit control-path exchange, diagnostic
details in existing responses or errors, and capability advertisement during
session setup. It deliberately leaves a wire format for later working-group
discussion.

This is an individual draft workspace. It does not imply working-group
adoption. See the issue and working-group discussion for current coordination
and authorship decisions.

## Build

This repository uses the IETF Internet-Draft template workflow. See the
[template setup instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md)
for local requirements. The GitHub Actions workflows are configured to publish
the editor's copy when changes are pushed to the repository.
