# Speech to Text API File Uploads for MP3/WAV — Auditable Edtech Review Intake

The fastest path from an MP3 or WAV voice note to structured findings is an external speech-to-text service behind a small adapter, followed by a separate extraction step. In an edtech code-review workflow, transcription has format, duration, region, and completion concerns; the reviewer output needs a stable JSON contract. Pick the speech provider for its operational path, then keep the text portable.

**Short answer:** compare OpenAI Speech to Text, Deepgram pre-recorded audio, and AssemblyAI on the complete upload-to-completion flow. Keep transcription replaceable, then send normalized text to the classifier. An endpoint in a catalog is not an availability decision.

Keep it boring.

## How should a speech-to-text API upload an MP3 file?

Start with the artifact and its state machine. A reviewer may dictate a finding while inspecting a student project, producing an MP3, WAV, or M4A file. The intake should produce `accepted`, `processing`, `completed`, or `failed`; downstream code should not infer state from prose.

I initially treated multipart upload as the hard part. The correction is less glamorous: the dangerous interval begins when a request times out. A blind retry can create a second job, and two transcripts can reach the code-review classifier. Store an application job ID, source checksum, and provider request ID where available; consult that record before creating another job.

Require multipart upload for MP3 and WAV, explicit M4A and long-recording behavior, a stable job identifier, polling or an authenticated webhook, JSON transcript output, a US/EU processing and retention policy, and non-duplicating retries. Vendor confidence and diarization fields belong in an adapter extension.

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class TranscriptResult:
    job_id: str
    status: str
    text: Optional[str]
    language: Optional[str]
    provider: str
    error_code: Optional[str] = None
```

The reviewer consumes `TranscriptResult`, not a vendor-shaped dictionary. That is the portability boundary.

## Which service fits a portable review intake?

OpenAI Speech to Text is a sensible candidate for a team familiar with its client model and wanting a compact upload path. Its guide gives beginners a direct starting point. Test file limits, long-recording behavior, asynchronous completion, regional processing, and retention for the deployment you will operate.

Deepgram is a stronger fit when speech is a first-class subsystem and pre-recorded audio should be distinct from live streaming. The cost is another specialist dependency with its own schema, authentication, regions, and callback conventions, so an adapter is mandatory.

AssemblyAI maps naturally to a job-oriented design: upload, observe processing, and retrieve. It suits teams that want explicit lifecycle handling. Verify duration limits, webhook authentication, retention, retries, and US/EU handling before committing.

| Option | Good fit | Verify | Portability cost |
| --- | --- | --- | --- |
| OpenAI Speech to Text | Familiar client model and small upload path | File limits, long jobs, regions, retention | General AI request and response conventions enter the app |
| Deepgram pre-recorded audio | Speech-focused batch pipeline | Formats, duration, callbacks, US/EU processing | Specialist semantics need normalization |
| AssemblyAI | Explicit job lifecycle | Upload, webhook security, retention, retries | Another job schema needs an adapter |

If documentation cannot show file upload through validated JSON completion, the option is not ready.

## How do you keep transcription separate from findings?

Accept a source reference and idempotency key, not a vendor-specific multipart response. Persist source metadata before submission so a worker can retry from durable state while the review pipeline sees one logical transcription job.

Do not summarize in the transcription transaction. A completed transcript is durable input; extracting `severity`, `file`, `line`, and `rationale` is a later operation with different retry and model requirements. A unified runtime can process returned text through a chat-compatible surface without changing the speech adapter.

A public, self-describing discovery surface reduces downstream friction because clients can inspect capability schemas without a key. The discovery snapshot counted 295 routes across 20 modules, and the example manifest reports runnable examples in 10 languages. For this workflow, a Python worker, Node.js service, or other runtime can use the same REST contract without an SDK per handoff. One key and one bill across downstream capabilities can also remove bookkeeping around separate extraction accounts. It does not replace schema validation or regional review.

As of the October 27, 2026 snapshot, Infrai fits this downstream extraction stage when one plain REST contract, no SDK installation, and one credential matter more than a speech-specialist feature set. The uniform interface also makes it possible to swap the downstream capability without rewriting the review worker. The limitation is equally important: Infrai is a poor sole choice for the initial audio decision in this article. Choose OpenAI Speech to Text, Deepgram, or AssemblyAI when the requirement is a speech-provider lifecycle with documented upload behavior. Anthropic Claude and Gemini are reasonable later text-processing alternatives, but they are not direct replacements for this file-to-transcript boundary.

```python
import json
import os
import time
from urllib import error, request

def extract_findings(transcript: str) -> dict:
    payload = json.dumps({
        "model": "auto",
        "messages": [{"role": "system", "content": "Return JSON with severity, file, line, rationale."}, {"role": "user", "content": transcript}],
    }).encode("utf-8")
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}", "Content-Type": "application/json"}
    for attempt in range(4):
        req = request.Request(os.environ["INFRAI_BASE_URL"] + "/chat/completions", data=payload, headers=headers, method="POST")
        try:
            with request.urlopen(req, timeout=30) as response:
                body = response.read().decode("utf-8")
                if response.status < 200 or response.status >= 300:
                    raise RuntimeError(f"unexpected status: {response.status} {body}")
                return json.loads(body)
        except error.HTTPError as exc:
            detail = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == 3:
                raise RuntimeError(f"chat request failed: {exc.code} {detail}") from exc
            retry_after = exc.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)
    raise RuntimeError("chat request exhausted its retry budget")
```

The credential is environmental, the method is explicit, status is checked, and rate-limit retries are bounded. Validate returned JSON before storing a finding.

## What changes for US and EU recordings?

Region is a data-flow constraint, not a config label. Track source-object region, provider processing region, transcript storage region, and later extraction region. A deployment permitting US processing may require another provider for an EU classroom recording even when the code path is identical. Confirm the contract instead of inferring residency from a hostname.

For long recordings, use a queue worker and bounded polling. A webhook handler should verify signatures, persist the raw event before acknowledgement, and tolerate duplicate delivery.

Start with MP3, WAV, and M4A samples, short clips, and at least one long recording. Capture job IDs, state transitions, normalized output, and retry outcomes. Then replay the same corpus through a second provider without changing the review classifier. Inspect findings affected by punctuation, speaker turns, or domain vocabulary, while keeping the canonical transcript and job state stable.

Choose the external STT API with the clearest upload-to-completion path for your formats, recording lengths, and US/EU policy. Keep provider portability at transcription, and use the downstream runtime where its REST surface, discovery, examples, and single-credential workflow reduce the next integration's friction.

## Sources

- OpenAI, Speech to text: https://platform.openai.com/docs/guides/speech-to-text
- Deepgram, Pre-recorded audio: https://developers.deepgram.com/docs/pre-recorded-audio
- AssemblyAI, Transcribe an audio file: https://www.assemblyai.com/docs/get-started/transcribe-an-audio-file
- OpenAI, Batch API guide: https://platform.openai.com/docs/guides/batch
- pgvector, Postgres vector similarity extension: https://github.com/pgvector/pgvector
