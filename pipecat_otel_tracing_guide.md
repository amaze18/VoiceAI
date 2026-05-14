# Pipecat + OTel Tracing: Langfuse & Hamming Integration Guide

> **Why this exists:** Pipecat's raw OTel spans tell you *how long* each service took,
> but not *what went wrong* semantically. Langfuse adds LLM-level conversation
> visibility. Hamming adds voice-specific QA and production alerting.

---

## How Pipecat Structures Its Traces (Baseline)

Before plugging in any backend, understand what Pipecat emits natively:

```
Conversation (conversation-uuid)        ← one per full session
├── turn-1
│   ├── stt_deepgramsttservice          ← duration, TTFB
│   ├── llm_openaillmservice            ← tokens, latency
│   └── tts_cartesiattsservice          ← characters, TTFB
└── turn-2
    ├── stt_deepgramsttservice
    ├── llm_openaillmservice
    └── tts_cartesiattsservice
```

**What's missing by default:**
- No conversation content (messages, transcripts) at the trace level
- No semantic scoring (did the agent answer correctly?)
- No ASR confidence scores
- No alerting on quality regressions

---

## Option 1: Langfuse (LLM observability focus)

### Step 0 — Install dependencies

```bash
pip install "pipecat-ai[tracing]"
pip install opentelemetry-exporter-otlp-proto-http
```

### Step 1 — Get & encode your Langfuse API keys

```bash
# Grab keys from https://cloud.langfuse.com → Project Settings
echo -n "pk-lf-YOUR_PUBLIC_KEY:sk-lf-YOUR_SECRET_KEY" | base64
# → Copy the output, you'll need it below
```

### Step 2 — Configure environment variables

```bash
# .env
OTEL_EXPORTER_OTLP_ENDPOINT="https://cloud.langfuse.com/api/public/otel"
# US region: https://us.cloud.langfuse.com/api/public/otel
OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic <BASE64_KEY>,x-langfuse-ingestion-version=4"

ENABLE_TRACING=true
# OTEL_CONSOLE_EXPORT=true  # uncomment for local debug output
```

### Step 3 — Wire tracing into your Pipecat bot

```python
# bot.py
import os
from dotenv import load_dotenv

from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from pipecat.utils.tracing.setup import setup_tracing
from pipecat.pipeline.pipeline import Pipeline
from pipecat.pipeline.task import PipelineParams, PipelineTask
from pipecat.services.deepgram.stt import DeepgramSTTService
from pipecat.services.openai.llm import OpenAILLMService
from pipecat.services.cartesia.tts import CartesiaTTSService

load_dotenv()

# ── 1. Initialize OTel → Langfuse ─────────────────────────────────────────────
# OTLPSpanExporter reads OTEL_EXPORTER_OTLP_ENDPOINT and
# OTEL_EXPORTER_OTLP_HEADERS automatically from env
exporter = OTLPSpanExporter()

setup_tracing(
    service_name="my-voice-agent",
    exporter=exporter,
    console_export=bool(os.getenv("OTEL_CONSOLE_EXPORT")),
)

# ── 2. Build your pipeline (normal Pipecat code) ───────────────────────────────
stt = DeepgramSTTService(api_key=os.getenv("DEEPGRAM_API_KEY"))
llm = OpenAILLMService(api_key=os.getenv("OPENAI_API_KEY"), model="gpt-4o")
tts = CartesiaTTSService(api_key=os.getenv("CARTESIA_API_KEY"), voice_id="...")

pipeline = Pipeline([stt, llm, tts])

# ── 3. Enable tracing in PipelineTask ─────────────────────────────────────────
task = PipelineTask(
    pipeline,
    params=PipelineParams(
        allow_interruptions=True,
        enable_metrics=True,        # Required — enables TTFB & usage metrics
    ),
    enable_tracing=True,            # Turns on OTel span emission
    enable_turn_tracking=True,      # Groups spans per conversational turn
    conversation_id="session-001",  # Optional; auto-UUID if omitted
    additional_span_attributes={    # Attach any custom metadata
        "customer.tier": "premium",
        "deployment.region": "in-south",
    },
)
```

### Step 4 — Add conversation content to traces (critical fix)

By default the trace has no input/output text. Patch it:

```python
# patch_trace.py — call this BEFORE setup_tracing()
def patch_trace_input_output():
    """
    Pipecat traces don't include conversation content by default.
    This monkey-patches the LLM span decorator to write:
      - langfuse.trace.input  → messages from the first LLM call
      - langfuse.trace.output → response from every LLM call (last wins)
    """
    from pipecat.utils.tracing import service_decorators

    original = service_decorators.add_llm_span_attributes
    first_call = [True]

    def patched(span, *args, **kwargs):
        original(span, *args, **kwargs)

        # Capture the prompt messages as the trace input (first turn only)
        if first_call[0] and kwargs.get("messages"):
            import json
            span.set_attribute(
                "langfuse.trace.input",
                json.dumps(kwargs["messages"])
            )
            first_call[0] = False

        # Capture every LLM response as the running trace output
        if kwargs.get("completion"):
            span.set_attribute(
                "langfuse.trace.output",
                kwargs["completion"]
            )

    service_decorators.add_llm_span_attributes = patched


# Usage — call before setup_tracing()
patch_trace_input_output()
```

### Step 5 — Rename traces (optional but recommended)

```python
# Pipecat names every trace "conversation" by default.
# Use span processors to inject a human-readable name.
from opentelemetry.sdk.trace import SpanProcessor

class TraceRenameProcessor(SpanProcessor):
    """Renames the root conversation span with caller ID / session info."""

    def __init__(self, name: str):
        self.name = name

    def on_start(self, span, parent_context=None):
        # Only rename the root span (no parent)
        if span.name in ("conversation",) and not span.parent:
            span.update_name(self.name)
            span.set_attribute("langfuse.trace.name", self.name)

    def on_end(self, span): pass
    def shutdown(self): pass
    def force_flush(self, timeout_millis=30000): return True


# Wire it in alongside the exporter:
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

provider = TracerProvider()
provider.add_span_processor(TraceRenameProcessor("support-call-+91XXXXXXXX"))
provider.add_span_processor(BatchSpanProcessor(exporter))
```

### What you see in Langfuse

| View | What it shows |
|---|---|
| Trace timeline | Full conversation with per-turn STT/LLM/TTS spans |
| Latency waterfall | Exactly where time is spent (e.g. LLM is 600ms of 800ms total) |
| Token usage | Input/output tokens per turn, aggregated per session |
| TTFB metrics | Time-to-first-byte for TTS and LLM streaming |
| Input/Output | Actual transcript + LLM response (after Step 4 patch) |

**Reference:** https://langfuse.com/integrations/frameworks/pipecat
**Example repo:** https://github.com/pipecat-ai/pipecat-examples/tree/main/open-telemetry/langfuse

---

## Option 2: SigNoz (self-hosted OTel backend, infra-grade)

Good if your team already runs an OTel stack or wants to self-host everything.

### Step 1 — Run SigNoz locally (or use SigNoz Cloud)

```bash
git clone https://github.com/SigNoz/signoz.git
cd signoz/deploy
docker-compose up -d
# UI at http://localhost:3301
```

### Step 2 — Wire Pipecat to SigNoz via gRPC exporter

```python
# Uses the gRPC exporter instead of HTTP
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from pipecat.utils.tracing.setup import setup_tracing

exporter = OTLPSpanExporter(
    endpoint="http://localhost:4317",  # SigNoz OTLP gRPC endpoint
    insecure=True,
)

setup_tracing(
    service_name="pipecat-voice-agent",
    exporter=exporter,
)
```

### Step 3 — Add structured logging (pairs well with traces)

```python
# structured_logging.py
import sys
from loguru import logger
from opentelemetry import trace

# Configure JSON logging that embeds the active trace/span ID
def add_trace_context(record):
    current_span = trace.get_current_span()
    ctx = current_span.get_span_context()
    record["extra"]["trace_id"] = format(ctx.trace_id, "032x") if ctx.is_valid else "N/A"
    record["extra"]["span_id"]  = format(ctx.span_id, "016x") if ctx.is_valid else "N/A"

logger.remove()
logger.add(
    sys.stdout,
    format='<green>{time:YYYY-MM-DD HH:mm:ss.SSS}</green> | <level>{level}</level> | '
           'trace={extra[trace_id]} span={extra[span_id]} | {message}',
    level="INFO",
)
logger.configure(patcher=add_trace_context)
```

---

## Option 3: Hamming (voice-native QA + production alerting)

Hamming sits on top of your OTel traces and adds semantic evaluation —
the layer that answers "did the agent actually do the right thing?"

### How it works

```
Pipecat Agent
     │
     │  OTel spans (conversation/turn/stt/llm/tts)
     ▼
Hamming Platform
     ├── Replayable call traces (linked to audio)
     ├── ASR accuracy scoring per turn
     ├── LLM-as-judge intent evaluation
     ├── Task completion scoring
     └── Regression alerts on prompt changes
```

### Integration pattern

Hamming ingests your Pipecat OTel traces and correlates them with call recordings.
Point your OTLP exporter at Hamming's endpoint:

```python
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from pipecat.utils.tracing.setup import setup_tracing

# Get endpoint + API key from https://app.hamming.ai → Settings
exporter = OTLPSpanExporter(
    endpoint="https://app.hamming.ai/api/otel/v1/traces",
    headers={"Authorization": f"Bearer {os.getenv('HAMMING_API_KEY')}"},
)

setup_tracing(service_name="my-pipecat-agent", exporter=exporter)
```

### Add call metadata for richer Hamming traces

```python
task = PipelineTask(
    pipeline,
    params=PipelineParams(enable_metrics=True),
    enable_tracing=True,
    enable_turn_tracking=True,
    conversation_id=call_id,           # Ties the trace to a specific call
    additional_span_attributes={
        "call.phone_number": caller_id,
        "call.direction": "inbound",
        "agent.version": "v2.1.0",     # Crucial for regression detection
        "agent.prompt_hash": prompt_hash,
    },
)
```

### What Hamming gives you over raw OTel

| Raw Pipecat OTel | + Hamming |
|---|---|
| STT span duration | + ASR confidence score, did transcript match intent? |
| LLM span duration | + Did the LLM response complete the task? |
| TTS span duration | + Was the audio quality acceptable? |
| No semantic scoring | + LLM-as-judge scores per turn |
| Manual log correlation | + Replayable trace linked to actual audio recording |
| No regression detection | + Alert when new prompt hurts task completion rate |

**Reference:** https://hamming.ai/resources/monitor-pipecat-agents-production-logging-tracing-alerts

---

## Sending Traces to Two Backends Simultaneously

Want Langfuse for LLM debugging AND SigNoz for infra dashboards?
Use a `MultiSpanExporter`:

```python
from opentelemetry.sdk.trace.export import SimpleSpanProcessor
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry import trace

from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter as HTTPExporter
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter as GRPCExporter

langfuse_exporter = HTTPExporter(
    endpoint="https://cloud.langfuse.com/api/public/otel",
    headers={"Authorization": f"Basic {os.getenv('LANGFUSE_B64_KEY')}"},
)

signoz_exporter = GRPCExporter(
    endpoint="http://localhost:4317",
    insecure=True,
)

provider = TracerProvider()
provider.add_span_processor(SimpleSpanProcessor(langfuse_exporter))
provider.add_span_processor(SimpleSpanProcessor(signoz_exporter))
trace.set_tracer_provider(provider)

# Then use setup_tracing with no exporter arg — provider is already set globally
```

---

## Debugging Checklist: When Traces Still Aren't Helping

| Symptom | Fix |
|---|---|
| Traces arrive but no input/output text | Apply the `patch_trace_input_output()` patch (Step 4) |
| All traces named "conversation" | Add `TraceRenameProcessor` with caller/session info |
| STT span looks fine but agent responded wrong | Add ASR confidence attribute manually; consider Hamming for semantic eval |
| Latency spikes visible but cause unclear | Check `stt` vs `llm` span duration ratio — LLM context window accumulation is usually the culprit |
| Traces don't arrive in Langfuse | Verify base64 encoding includes colon between public:secret key |
| Turn spans missing | Ensure `enable_turn_tracking=True` AND `enable_metrics=True` both set |
