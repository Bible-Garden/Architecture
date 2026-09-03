# ADR-0001: Require separate explicit consent for Lampada AI processing

- Status: Accepted
- Date: 2026-09-03
- Participants: product owner, project team

## Context

Lampada processes prayer topics, written answers and voice recordings. This
content can reveal religious beliefs and other highly sensitive facts even
without an account or a name. It crosses two repositories: `Lampada-Mobile`
builds the requests and `Bible-API` forwards them to the external AI provider.

The old mobile implementation had one `share_answers` boolean. A missing value
meant `true`, so a new installation could send typed answers and transcripts
without a decision. Pressing `Transcribe` started an upload without a separate
informed decision about Google Gemini. The prayer topic also left the device for
the first generated question before either control was relevant.

Production used the unpaid Gemini API tier. Google's terms allow unpaid content
to be used for product improvement and human review and tell developers not to
submit sensitive, confidential or personal information to unpaid services. That
contract is incompatible with prayer content.

## Decision

1. Keep three independent, versioned consent records:
   - core prayer AI: sending the topic for questions and scripture selection;
   - answer context: sending typed answers and completed transcripts;
   - audio transcription: sending a selected M4A recording.
2. Each record is `undecided`, `allowed` or `denied`. Missing, malformed, legacy
   permissive or obsolete values never mean `allowed`.
3. Ask at the first relevant action, immediately before content would cross the
   boundary. Name Google Gemini, the content category, the purpose, the Lampada
   server path and the fact that Bible API does not persist the content.
4. A refusal leaves the prayer usable. Core questions use local curated pools;
   scripture selection may use the non-contextual safe pool. Refusing answer
   context still permits topic-only AI when core consent is allowed. Refusing
   transcription leaves the local recording intact.
5. Expose all decisions in settings. Withdrawal is persisted before the next
   request can be built.
6. Migrate an old `share_answers=0` to `denied`; migrate `1` or a missing value
   to `undecided`. Core AI and transcription start as `undecided` on existing
   installations.
7. Use only a paid Gemini API project in production. Keep Gemini developer
   logging and data sharing disabled. This is a release gate.
8. Re-consent when the provider, legal processor, training or human-review use,
   retention, or processing jurisdiction changes materially. A model-only
   change under the same data contract does not invalidate consent.
9. Keep Lampada Mobile, Bible API, the public Privacy Policy and App Store
   Privacy answers aligned with the detailed rules in
   [`../privacy/lampada-data-processing.md`](../privacy/lampada-data-processing.md).

## Options considered

### Keep the enabled-by-default answer toggle and treat use as consent

Rejected. A default, silence and general app use do not prove a specific informed
decision and cannot distinguish the three purposes.

### Use one consent for every AI feature

Rejected. A person may accept topic-based questions while keeping answers local,
or request one transcription without allowing its text into later prompts.

### Keep the unpaid Gemini tier after adding UI consent

Rejected. Consent does not remove Google's warning against submitting sensitive
data to unpaid services, and the content could still be used for product
improvement and human review.

### Remove all external AI processing

Not chosen. Local fallbacks keep the prayer usable, while explicit independent
choices preserve the AI features for people who want them.

## Consequences

- A fresh or upgraded installation performs no external prayer-content request
  until the relevant decision is made.
- Existing permissive state is deliberately not grandfathered.
- Request builders and network entry points need hard gates, not UI-only checks.
- Consent records need a notice and provider-contract version.
- Operating the production AI surface now has a direct cost and a release check.
- The Privacy Policy must describe Google Gemini, the three purposes, provider
  processing and Bible API's non-persistence accurately.

## References

- [Detailed data processing rules](../privacy/lampada-data-processing.md)
- [Lampada-Mobile](https://github.com/BibleGarden/Lampada-Mobile)
- [Bible-API](https://github.com/BibleGarden/Bible-API)
- [ClickUp: rules and provider change](https://app.clickup.com/t/86cbcunjp)
- [ClickUp: topic and answer consent](https://app.clickup.com/t/86cbcunkb)
- [ClickUp: audio transcription consent](https://app.clickup.com/t/86cbcunm6)
- [Gemini API Additional Terms of Service](https://ai.google.dev/gemini-api/terms)
- [Gemini API data logging and sharing](https://ai.google.dev/gemini-api/docs/logs-policy)
