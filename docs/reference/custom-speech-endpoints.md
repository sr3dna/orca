# Custom OpenAI-compatible speech endpoints

Orca's voice dictation normally runs one of the bundled on-device models or the
OpenAI cloud models. This document describes the third option: pointing dictation
at **any OpenAI-compatible `/audio/transcriptions` server** the user controls —
for example a self-hosted `faster-whisper` worker, Speaches, LocalAI, or a
whisper.cpp server.

## Why an OpenAI-shaped endpoint and not a bespoke protocol

There is no W3C/IETF standard for speech-to-text HTTP APIs. The de-facto standard
is OpenAI's transcription API, which a large number of self-hosted servers
implement:

```
POST /v1/audio/transcriptions
Content-Type: multipart/form-data
  file=<audio>
  model=<string>       # optional per server
  language=<ISO code>  # optional
  response_format=json|text|verbose_json|srt|vtt
→ { "text": "..." }
```

Supporting that single shape reaches the widest set of servers with one code path.
Orca reuses its existing OpenAI transcription client
(`src/main/speech/openai-transcription-client.ts`); only the resolved
`{ url, apiKey, apiModel, language }` target differs between the built-in cloud
models and a user endpoint.

## What the user configures

Stored in `~/.orca/custom-stt-endpoint.json` (base URL + model + optional
language) with the bearer token sealed separately in
`~/.orca/custom-stt-token.enc` through the shared secret store.

| Field | Required | Notes |
| ----- | -------- | ----- |
| Base URL | yes | `http://` or `https://`. `/audio/transcriptions` is appended unless already present. |
| Model | yes | Sent verbatim as the multipart `model` field. |
| Language | no | ISO code sent as `language`; empty means auto-detect. |
| API key | no | Bearer token; empty for servers without auth. |

The endpoint is considered "ready" whenever a base URL and model are saved; there
is no API-key gate (unlike the OpenAI cloud models). Dictation audio is only sent
to this server while the **Custom endpoint** model is selected.

## Model field: discovered suggestions

After the user types a base URL (debounced), Orca asks the endpoint what it
supports and turns the answers into suggestions for the Model field. There is no
single standard for this either, so the probe tries, in order:

1. `GET <root>/v1/models` — the OpenAI shape (`{ data: [{ id }] }`).
2. `GET <root>/health` — leaner self-hosted servers (e.g. a `faster-whisper`
   worker) expose `supportedModels` here instead.
3. `GET <root>/models` — a common alternative.

Where `<root>` strips a trailing `/v1` or `/audio/transcriptions` so any base URL
form works. A miss is harmless: the field stays free text and a model is **not**
required to press **Test**, which only needs a base URL to prove reachability.

As with language, the discovered names are suggestions, not a whitelist — the
user can always type a model the endpoint did not advertise.

## The language field: free text, with suggestions

The language field is deliberately **free text with a suggestion list**, not a
fixed dropdown, for two reasons:

1. **The accepted set is server/model specific.** Whisper accepts ~99 codes; a
   different server or fine-tune may accept a different set. No static list can
   cover every endpoint.
2. **A strict ISO-639-1 list would exclude valid codes.** `yue` (Cantonese) and
   `haw` (Hawaiian) are ISO-639-3 three-letter codes that Whisper uses. A
   2-letter-only dropdown would silently drop them — and Cantonese is a
   first-class case for at least one local worker, which injects a Cantonese
   initial prompt when `language=yue`.

The suggestions in `src/shared/speech-language-options.ts` are therefore
*examples*, not a whitelist. The user can type any code their server accepts.

### Why mistakes are harmful

A wrong language value is not harmless. Servers differ, but a strict server —
such as a `faster-whisper` worker that validates against its accepted list —
rejects the whole request:

| Input | Typical server behaviour | Result |
| ----- | ------------------------ | ------ |
| empty | auto-detect | works |
| `en`, `yue`, `zh` | pin / prompt | works |
| `english` | not a code → 4xx/5xx | **dictation fails** |
| `zh-CN` | region tag → not a code | **dictation fails** |

Because the failure mode is a hard error rather than a silent fallback, the UI
must not let a typo silently brick dictation.

## Test and the Save gate

The **Test** button probes the saved *or on-screen draft* values by POSTing a
~0.1 s silent WAV and classifies the outcome. That classification drives how
strict the Save gate is, so a typo is caught without locking users out when their
server is merely offline:

| Outcome | Meaning | Save allowed? |
| ------- | ------- | ------------- |
| `ok` | 2xx | yes |
| `auth` | 401/403 | yes (server is reachable; fix the token later) |
| `rejected` | any 4xx, or a 5xx that returns a body | **blocked** — the parameters would not work; a "Save anyway (advanced)" escape hatch remains |
| `transport` | unreachable / bare 5xx / timeout | yes (may just be offline) |
| `invalid` | malformed draft (bad URL, empty fields) | **blocked** |

Two implementation notes:

- The `rejected` class deliberately spans both 4xx and explained 5xx because
  servers disagree on the status for a bad parameter: a `faster-whisper` worker
  answers an invalid language with **500**, not 400. Classing only 4xx as a
  rejection would miss exactly the field this feature must guard.
- A silent probe clip can be rejected by some servers even when the config is
  otherwise fine (e.g. a "too short audio" 4xx). That is why a rejection is a
  warning with an explicit override rather than an absolute block.
- `fetch` collapses transport failures into `"fetch failed"`; the actionable
  cause (`ECONNREFUSED`, TLS, DNS) is on `error.cause` and is surfaced one level.
  A single retry follows, because undici reuses keep-alive connections that
  servers such as uvicorn close when idle, which fails a POST once.

## Security posture

- The endpoint URL is user-controlled, so only `http`/`https` are accepted and
  the value is validated before any request. The desktop app makes the request
  from the user's own machine, so this is not a server-side SSRF surface.
- The bearer token is never logged, and the Test request never follows a redirect
  with credentials.
- Self-hosted workers are commonly unauthenticated; bind them to loopback
  (`127.0.0.1`) rather than `0.0.0.0`.

## Related code

- `src/main/speech/custom-stt-endpoint-store.ts` — persistence + URL resolution
- `src/main/speech/custom-stt-endpoint-test.ts` — probe + outcome classification
- `src/main/speech/stt-transcription-target.ts` — resolves cloud vs custom target
- `src/main/speech/openai-transcription-client.ts` — the shared HTTP client
- `src/shared/speech-language-options.ts` — language suggestions
- `src/renderer/src/components/settings/CustomSttEndpointDialog.tsx` — settings UI
