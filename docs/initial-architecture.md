# Initial architecture: Parsec Alexa Skill

Status: discovery and design baseline. No production changes are authorized by this document.

## Product scope

The first Alexa version is a private German-language skill for one Echo device per voice request. It should support:
- natural-language and semantic music search;
- exact title, artist and album lookup;
- similar-track discovery;
- playlist playback (playlist ownership/source is still an open product decision);
- direct playback from the Parsec Gateway to the requesting Echo.

Multi-room playback is explicitly out of scope for the first version.

## Existing Parsec capabilities

### Search service
- LAION-CLAP text embeddings and cosine similarity are already implemented.
- Audio embeddings are persisted in SQLite, with 512 dimensions per track.
- The current public search route is semantic only: `POST /search`.
- The current index is keyed by relative music path and stores vectors plus source revision metadata.
- There is currently no explicit metadata lookup endpoint and no track-to-track nearest-neighbour endpoint.

### Gateway
- `POST /api/v1/search` exposes semantic search and maps paths to Gateway track IDs.
- `POST /api/v1/tracks/{trackId}/stream-ticket` issues a short-lived stream ticket.
- `GET /api/v1/stream/{ticketId}` streams audio and supports HTTP byte ranges.
- The current track registry and stream-ticket registry are in-memory. Existing default lifetimes (30 minutes for track references and 2 minutes for stream tickets) must be reviewed for Alexa playback and queue behaviour.
- The Gateway already owns the private NAS/audio-node transport and should remain the authority for resolving and streaming tracks.

### Worker
- The existing CPU/GPU worker pipeline generates audio embeddings and submits them through the Gateway.
- Similar-track search can reuse those vectors; it does not require another model inference per query once an anchor track is known.

## Proposed responsibility boundaries

### Alexa Skill
- German utterance and intent handling.
- Resolve the current Alexa device/request context.
- Start, pause, resume, stop, next and previous playback.
- Translate Gateway responses into Alexa speech and AudioPlayer directives.
- No direct NAS access, no audio proxying, and no embedding/database ownership.

### AWS Lambda
- Thin request handler for the Alexa Skill.
- Validate Alexa requests using the official ASK SDK/request verification.
- Call the Gateway over HTTPS using a dedicated, narrowly scoped skill credential.
- Never use the Gateway admin token or worker/audio-node credentials.
- Do not store music metadata or playlist state in Lambda memory.

### Gateway
- Authenticate the Skill integration.
- Own playback sessions and queues, scoped to the Alexa device/user context.
- Resolve search results to track IDs.
- Issue fresh stream tickets when Alexa needs a URL, rather than relying on a ticket created at the start of a long queue.
- Own durable playlist state if the product decision is to make Parsec playlists independent of myMedia.

### Search
- Add deterministic metadata search for title, artist and album (semantic CLAP search is not a substitute for exact lookup).
- Add nearest-neighbour track similarity using stored audio vectors.
- Keep Search internal/API-only; do not publish a Search UI.

## Playback design constraints

- Alexa fetches the audio stream directly from the Gateway; Lambda must not proxy audio bytes.
- Preserve HTTPS on port 443, valid publicly trusted TLS, correct audio content type and byte-range behaviour.
- Test the actual NAS formats/codecs against Echo compatibility; do not assume every indexed format can be played directly.
- Use a playback session/queue in the Gateway. Alexa AudioPlayer events such as PlaybackNearlyFinished should trigger preparation of the next item.
- Issue stream tickets just in time for each track and verify ticket expiry against Alexa's range/retry behaviour.
- Keep session state durable enough to survive Lambda cold starts and Gateway restarts. Do not rely on process-local Lambda state.

## Open decisions

1. **Playlist ownership/source:** decide whether existing myMedia playlists must be imported/synchronised, or whether Parsec should manage independent playlists. Do not implement playlist storage until this is decided.
2. **Identity and access:** decide whether this private single-user skill needs Alexa account linking, or whether a dedicated skill credential bound to the private skill is sufficient. In either case, never expose admin credentials.
3. **Supported audio formats:** inspect the real library formats and test Echo playback before selecting any transcoding approach.
4. **Session identity:** define a stable, privacy-conscious key for separating queues by requesting Echo/device and account context.

## Delivery sequence

1. Search API: metadata lookup and similar-track endpoint, with tests and a stable release.
2. Gateway API: authenticated Skill boundary, playback session/queue, just-in-time stream tickets, tests and a stable release.
3. Alexa Skill: German intents, AudioPlayer event handling, Lambda packaging/deployment workflow, and simulator/device tests.
4. End-to-end validation on one Echo.
5. Production deployment only after stable releases and the official deploy workflow are ready; no ad-hoc production image rebuilds.

## Cost guardrails

- Prefer one Lambda function and existing Parsec infrastructure.
- Avoid introducing always-on AWS compute, managed databases or paid secrets services without a demonstrated need.
- The existing 1 EUR monthly AWS budget is an alert, not a hard spending cap. Verify current AWS pricing and free-tier eligibility before provisioning billable resources.
- No AWS runtime resources have been provisioned as part of this design baseline.

## Non-goals for the first version

- Multi-room playback.
- Public Search UI.
- Direct Alexa-to-NAS access.
- Replacing the existing worker/embedding pipeline.
