---
name: jellypod-api
description: >-
  How to use the Jellypod API to create AI-powered podcasts programmatically —
  creating hosts, uploading sources, generating episodes, and publishing. Use
  this skill whenever someone wants to build with the Jellypod API, integrate
  Jellypod into an application, generate podcast content programmatically,
  automate podcast production, or interact with any Jellypod endpoint. Also use
  when someone mentions AI podcasts, podcast generation APIs, or turning text
  into podcast episodes via code.
---

# Jellypod API

Jellypod is an AI podcast studio. Users describe an episode in plain language, attach source material (URLs, PDFs, YouTube videos, text), and Jellypod researches the topic, writes a multi-host script, and produces full audio. The API exposes that same pipeline so you can create AI hosts, upload research sources, generate episodes from a prompt, and publish to Spotify, Apple Podcasts, YouTube, and a Jellypod-hosted website — all from code.

This skill is the high-level guide. For exact endpoint shapes, field constraints, and request/response schemas, always defer to the canonical references below — they change more often than this document.

## Canonical references (fetch these for specifics)

- **Overview docs (markdown):** <https://jellypod.com/docs/api.md>
- **Full docs index (markdown for agents):** <https://jellypod.com/llms.txt>
- **OpenAPI spec:** <https://jellypod.com/docs/api/openapi.yaml>
- **Human-readable docs:** <https://jellypod.com/docs/api>

When the user asks about a specific endpoint, parameter, response field, or error code, fetch `openapi.yaml` rather than guessing — the spec is the source of truth and is regenerated from the backend Zod schemas on every deploy.

## Base URL

```
https://api.jellypod.com/v1
```

## Authentication

Bearer token in the `Authorization` header:

```
Authorization: Bearer sk_live_...
```

Keys are organization-scoped. Users create and manage them at <https://jellypod.com/studio/settings/api-keys> (Studio → Settings → API Keys).

## Mental model

These are the entities you'll be coordinating. The specifics live in the OpenAPI spec; the relationships matter most:

- **Podcast** — A series container. Holds title, description, default hosts, language, and visibility. Every organization starts with a default podcast ("My First Podcast") you can use immediately without creating one.
- **Episode** — One entry inside a podcast. Has its own script, audio, video, and publish state. Episodes are generated from a `prompt` plus optional sources, then can be published or scheduled.
- **Host** — An AI persona that narrates episodes. Hosts have a name, backstory (10–3000 chars), optional title and personality, and a voice. A podcast can have multiple hosts who converse naturally.
- **Voice** — A TTS voice from the library (100+ professional voices across 30+ languages, plus clones). Voices belong to hosts; pick one with `GET /voices` filtered by `language` / `gender` / `voice_type`.
- **Source** — Reference material (URL, YouTube, text, or uploaded file) used as research context when generating episodes. Sources are processed asynchronously before they're usable.
- **Credits** — Org-scoped balance consumed by episode generation. Check via `GET /account`. Same credit costs as the studio.

## Async operations — the most important thing to internalize

Three endpoints are asynchronous and return `202 Accepted` with a resource id. The work happens in the background; you have to poll the corresponding `GET` to know when it's done.

| Async endpoint | Poll | Typical duration |
|---|---|---|
| `POST /sources` (URL, YouTube, file) | `GET /sources/{id}` until `status: completed` | seconds to ~1 min |
| `POST /podcasts/{id}/episodes/generate` | `GET /episodes/{id}` until `status: draft` and `audio_url` populated | 2–8 min |
| `POST /podcasts/generate` (batch) | Poll each child episode individually | 5–30 min total |

Poll every ~5 seconds. While generating, the episode response includes a `generation` object with `phase` and `progress_pct`. If something fails, status becomes `failed` (or `error` for sources) — surface the error to the user rather than retrying blindly.

There's a per-org concurrent limit of **5 episodes generating simultaneously**. Exceeding it returns `429` with code `concurrent_limit_exceeded`. Back off and retry, don't fan out.

## The typical recipe

For most "generate a podcast episode from code" use cases:

1. `GET /podcasts` — every org has a default podcast and two default hosts. If those work, skip to step 5.
2. `GET /voices` — browse the library by language/gender to pick voices.
3. `POST /hosts` — create one or more hosts with a name, backstory, and `voice_id`.
4. `POST /podcasts` — create the series, passing the host ids.
5. *(Optional)* `POST /sources` for each piece of research material, then poll `GET /sources/{id}` until `completed`.
6. `POST /podcasts/{id}/episodes/generate` with a `prompt` and optional `source_ids`.
7. Poll `GET /episodes/{id}` every 5s until `status: draft`.
8. `POST /episodes/{id}/publish` to release immediately, or pass `scheduled_time` to schedule.

For "spin up a whole show in one call" use `POST /podcasts/generate` instead of steps 4–7 — it creates the podcast, plans episode topics, and kicks off audio generation for up to 8 episodes. The initial response (~5–15s) contains the podcast plus LLM-generated episode titles; each episode then generates in the background.

## Pagination

List endpoints use cursor pagination: pass `cursor` and `limit` (1–100, default 20). Responses include `{ pagination: { has_more, next_cursor } }`. Pass `next_cursor` as `cursor` on the next call; stop when `has_more` is `false`.

## Errors

All errors share the same envelope:

```json
{
  "error": {
    "code": "validation_error",
    "message": "backstory must be at least 10 characters",
    "request_id": "req_abc123",
    "details": [{ "field": "backstory", "message": "must be at least 10 characters" }]
  }
}
```

`details` is populated for `validation_error`. Include `request_id` when reporting issues — it's the fastest way to find a request in our logs. Common codes: `bad_request`, `validation_error`, `unauthorized`, `insufficient_credits`, `not_found`, `unprocessable_entity`, `rate_limited`, `concurrent_limit_exceeded`, `internal_error`.

## Rate limits

Every response includes `RateLimit-Limit`, `RateLimit-Remaining`, and `RateLimit-Reset` headers. On `429`, honor `Retry-After`.

## Gotchas worth knowing up front

- **ID shapes vary by resource.** Podcast / episode / source ids are UUIDs, voice ids are integers, and host ids are short alphanumeric strings. Don't assume a single id format — pass whatever the create response returned.
- **Cover-image upload is raw bytes**, not multipart: `PUT /episodes/{id}/image` (or `/podcasts/{id}/image`) with `Content-Type: image/png | image/jpeg | image/webp` and the file in the body. Max 10 MB.
- **Source file uploads are multipart/form-data** — `file` plus an optional `title`. Max 10 MB. Supported: PDF, DOCX, PPTX, CSV, markdown, plain text.
- **Deletes are not always destructive.** Archiving a host (`DELETE /hosts/{id}`) keeps it linked to existing episodes; deleting a podcast removes its episodes irreversibly. Read the spec before wiring up cleanup logic.

## When in doubt

Fetch `https://jellypod.com/docs/api/openapi.yaml` and read the relevant operation. Every endpoint's request body, query params, response shape, and possible error codes are described there, and the spec is regenerated on every backend deploy so it never drifts.
