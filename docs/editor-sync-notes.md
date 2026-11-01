# Editor Sync Notes

Scratch notes on how the collaborative editor keeps clients in sync.

## Current picture

- Clients send local edits to the backend service.
- The service reconciles concurrent edits and broadcasts the merged state.
- Reconnect after disconnect should end up with the same state as a fresh load.

## Open questions

- What happens to cursors/presence during long reconnect windows?
- Do we need back-pressure on very large documents?

Update this file as decisions land in code; it is a working doc, not a spec.
