# Lampada data processing rules

- Status: Approved
- Approved: 2026-09-03
- Owner: product owner
- Applies to: `Lampada-Mobile`, `Bible-API`, production infrastructure and the
  external AI provider

This document is the normative privacy source for Lampada. Lampada prayer
content can reveal religious beliefs, health, relationships and other highly
sensitive facts. It must be treated as sensitive personal data even when the app
has no account and does not know the person's name.

## Principles

1. Data stays on the device unless a named feature needs an external transfer
   and the person has explicitly allowed that transfer.
2. Silence, a missing value, an old default and continuing to use the app are
   never consent.
3. Consent is purpose-specific, versioned, reversible and independent for the
   three AI flows defined below.
4. A refusal leaves a useful local degradation and does not block prayer, the
   journal, recordings or reminders.
5. The client sends the minimum content needed for the selected purpose. Bible
   API and its logs do not persist prayer content.
6. Production prayer content may be processed only through a paid AI service
   whose data terms have been reviewed and whose optional logging and data
   sharing are disabled.

## Data inventory

| Data | Device storage | Network use | Retention and deletion |
| --- | --- | --- | --- |
| Prayer topic, generated questions, typed answers and takeaway | `lampada.db` in Expo SQLite | The topic may be sent for core AI processing. Answers may be sent only under the separate answer-context consent. | Kept locally until the prayer or all local data is deleted. Bible API does not persist the content. |
| Voice recordings | M4A files in the app document directory; portable paths in SQLite | A selected file may be sent only for transcription and only after transcription consent. It is never part of question or scripture context as audio. | Kept locally until the recording, prayer or all local data is deleted. Bible API reads the upload in memory and does not persist it. |
| Transcripts | `recordings.transcript` in SQLite | Returned by transcription; later treated as an answer and sent as context only under answer-context consent. | Same local lifetime as the recording. Bible API does not persist the transcript. |
| Scripture responses, history and favourites | SQLite snapshots and canonical identifiers | Selection sends language, translation and exclusions. Topic and replies are included only when their consent gates allow them. | Local cache and history remain until local data is wiped; favourites remain until removed or wiped. Request-derived server data is not cached. |
| Scripture catalogue, text alignment and audio | Catalogue and selected snapshots may be cached locally; narration is streamed | Requests go to Bible API and contain no prayer content. | Subject to the local cache and favourites lifetime. |
| Consent, scripture and reminder settings | The `meta` table in SQLite | Consent values are not sent to the AI provider. | Until changed or all local data is wiped. |
| PIN protection | Salt, PIN hash and flags in Expo SecureStore | Never transmitted. | Until the lock is disabled or all local data is wiped. The PIN itself is never stored. |
| Reminder schedule | SQLite and operating-system local notifications | Never sent to a push service. | Until changed, disabled or local data is wiped. |
| Prayer days | `prayed_days` in SQLite | Never transmitted. | Deleting a prayer does not erase its historical day; a full wipe does. |
| Diagnostics | Local `lampada-diagnostics.log` | Not transmitted by the app. | Deleted by a full wipe; it must not contain prayer content. |
| API request metadata | No client analytics store | Bible API sees the endpoint and network peer. Private AI endpoints store endpoint, method, status, latency and an HMAC pseudonym with an empty user agent. | Raw Bible API statistics are deleted after 14 days; permanent daily aggregates contain counts only. Request and response bodies are never logged. |

## AI purposes and consent

Each consent record has three states: `undecided`, `allowed` and `denied`. It
also carries a notice version and the identity of the provider contract that was
approved. A missing, malformed or obsolete record is `undecided`.

### Core prayer AI

Purpose: generate guiding questions and select scripture using the prayer topic.

- Before the first request that would send the topic, explain that the topic is
  sent through Bible API to Google Gemini for these two purposes.
- While the state is `undecided`, no request containing prayer content starts.
- On `denied`, questions come from local curated pools. Scripture may use a
  non-contextual server safe pool, but the request contains neither topic nor
  replies.
- On `allowed`, only the topic and request context needed for the feature may be
  sent. This does not allow answer context or audio transcription.

### Answer context

Purpose: let later questions, the reflection question and scripture selection
take account of typed answers and completed transcripts.

- Ask separately before the first request that would need an answer.
- `user_replies` is absent from the serialized scripture request unless this
  state is `allowed`.
- Question prompts omit typed answers and transcripts unless this state is
  `allowed`.
- The scripture endpoint receives at most 10 replies, at most 1,000 characters
  per reply and at most 4,000 characters in total.
- A refusal still permits core AI processing of the topic if its separate state
  is `allowed`.

### Audio transcription

Purpose: turn one chosen voice recording into a verbatim transcript.

- Ask separately before the first upload. Explain that the M4A file is sent
  through Bible API to Google Gemini only for transcription.
- Pressing `Transcribe` is the feature request but is not by itself informed
  consent.
- On `undecided` or `denied`, the upload does not start. The local recording
  remains playable and deletable.
- Allowing transcription does not allow the resulting text to be used as answer
  context.

The allow and deny actions have equal visual weight. Settings expose all three
decisions and allow them to be changed or withdrawn. Withdrawal is persisted
before returning from the action and applies to the next request. An already
completed external request cannot be recalled.

## Existing installations

The old `share_answers` boolean did not record a notice version and treated a
missing value as permission:

- missing or `1` becomes answer-context `undecided`;
- an explicit `0` remains `denied`;
- core prayer AI and audio transcription start as `undecided` for every existing
  installation.

No legacy state silently produces `allowed` under the new contract.

## Current provider and operating requirements

The current external AI provider is Google Gemini, called by Bible API through
`generativelanguage.googleapis.com`. Bible API forwards content in memory and
does not store prayer topics, replies, recordings, transcripts or generated
responses. It records only the metadata described above.

Production must use a Gemini API project with active Cloud Billing. Google's
current terms say that unpaid Gemini API content may be used to improve Google
products and may be reviewed by people, and tell developers not to submit
sensitive, confidential or personal information to unpaid services. Paid
services do not use prompts or responses for product improvement, but Google may
retain them for a limited period for abuse prevention and legal obligations and
may process them in countries where Google or its agents operate.

Release requirements:

- active billing covers every model that can receive prayer content, including
  question generation, transcription, scripture rewrite, embeddings and rerank;
- developer-owned Gemini API logging is disabled;
- no logs or datasets are shared with Google and no sensitive request is sent as
  feedback;
- reverse-proxy and application logs contain no request or response bodies;
- the deployed provider, models and data settings match this document and the
  public Privacy Policy.

Optional Gemini developer logging can retain complete requests and responses for
up to 55 days and datasets can retain them without a fixed period. Lampada does
not enable either feature.

## Changing the provider or its terms

Before another provider or API route receives production content, record:

- legal entity and subprocessors;
- purposes and data categories;
- training and human-review rules;
- provider and developer-log retention;
- processing countries and transfer mechanism;
- deletion controls, security commitments and data-processing agreement;
- exact models and which stages receive raw or derived prayer content.

A change of provider, legal processor, training use, human review, retention or
processing jurisdiction invalidates the affected consent records and requires a
new notice and consent. A model change under the same processor and unchanged
data terms does not require new consent, but still needs quality, security and
documentation review.

Lampada Mobile disclosures, Bible API behaviour, the public Privacy Policy and
App Store Privacy answers are updated together. A provider change must not send
the same content to both old and new providers during migration. Public Lampada
documents live at `https://lampada.bible.garden/privacy` and
`https://lampada.bible.garden/support`; `bible.garden` remains the separate Bible
Garden site.

## Verification

- Mobile unit tests cover state parsing, legacy migration, every request gate
  and immediate withdrawal.
- Network tests assert that disallowed fields and uploads are absent, not empty
  placeholders.
- The release checklist verifies paid billing, disabled Gemini logging/data
  sharing, the deployed provider and matching public documents.
- A human verifies disclosure wording and App Store Privacy answers. This is an
  engineering policy, not a substitute for legal review.

## Sources

- [Gemini API Additional Terms of Service](https://ai.google.dev/gemini-api/terms), accessed 2026-09-03
- [Gemini API data logging and sharing](https://ai.google.dev/gemini-api/docs/logs-policy), accessed 2026-09-03
- [Gemini API billing](https://ai.google.dev/gemini-api/docs/billing), accessed 2026-09-03
- [Expo SQLite, SDK 57](https://docs.expo.dev/versions/v57.0.0/sdk/sqlite/)
- [Expo FileSystem, SDK 57](https://docs.expo.dev/versions/v57.0.0/sdk/filesystem/)
- [Expo SecureStore, SDK 57](https://docs.expo.dev/versions/v57.0.0/sdk/securestore/)
- [Expo Audio, SDK 57](https://docs.expo.dev/versions/v57.0.0/sdk/audio/)
- [Lampada-Mobile](https://github.com/BibleGarden/Lampada-Mobile), reviewed at `eddd9eab8754cd103dcf7cf774e41da417de65ed`
- [Bible-API](https://github.com/BibleGarden/Bible-API), reviewed at `fc0d24e5e647a68b93a4f542a60fc4483e55ea83`
