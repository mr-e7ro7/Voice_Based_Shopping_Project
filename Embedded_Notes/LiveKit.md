LiveKit is an open-source, real-time communication platform built on WebRTC. It started as infrastructure for audio and video conferencing, and has since become one of the primary frameworks for building production-grade AI voice agents, handling the full STT-LLM-TTS pipeline as a first-class concern.

---

## What It Is at the Core

LiveKit is an open-source project that provides scalable, multi-user conferencing based on WebRTC. The server is written in Go, using the Pion WebRTC implementation.

LiveKit's architecture is built around rooms, participants, and tracks, which are virtual spaces where users and agents connect and share media and data across web, mobile, and embedded platforms. Everything in LiveKit, whether a human user or an AI agent, is a participant in a room. This abstraction is what makes the transition from "conferencing tool" to "AI agent infrastructure" clean.

---

## The SFU Architecture

LiveKit is built around a **Selective Forwarding Unit (SFU)** rather than an MCU (Multipoint Control Unit). The distinction matters:

| | SFU | MCU |
|---|---|---|
| What it does | Forwards media streams selectively to each participant | Mixes all streams server-side into one |
| CPU cost | Low (no transcoding) | High (re-encodes everything) |
| Latency | Lower | Higher |
| Scalability | Better | Worse |

LiveKit's horizontal scaling capabilities address one of the most significant challenges in real-time communication: handling unpredictable traffic patterns. The system can deploy multiple SFU nodes with identical configurations, using Redis for peer-to-peer routing to ensure participants in the same session connect to the same node.

In LiveKit Cloud deployments, multiple SFU instances form a distributed mesh, allowing media servers to discover and relay content between regions. Participants automatically connect to the nearest instance, minimizing latency and packet loss.

---

## LiveKit Agents Framework

This is where LiveKit becomes relevant to AI systems. The agents SDK includes components for handling the core challenges of real-time voice AI, such as streaming audio through an STT-LLM-TTS pipeline, reliable turn detection, handling interruptions, and LLM orchestration. It supports plugins for most major AI providers.

### The Pipeline

A voice agent is built on three core components: a speech-to-text (STT) model that transcribes audio, a large language model that generates a response, and a text-to-speech (TTS) model that speaks it back.

```
User speaks
    │
    ▼
VAD (Voice Activity Detection)   ← detects speech vs silence
    │
    ▼
STT (e.g., Deepgram, AssemblyAI) ← audio to text, streaming
    │
    ▼
LLM (GPT-4o, Claude, etc.)       ← reasoning, tool calls
    │
    ▼
TTS (e.g., Cartesia, ElevenLabs) ← text to audio, streaming
    │
    ▼
Agent speaks back into the room
```

### Sequential vs Streaming Pipeline

In the simplest architecture, the voice agent waits for the user to finish speaking, transcribes the full utterance, sends it to the LLM, waits for the full LLM response, then passes it to TTS and plays it back. This is easy to build and reason about, but latency stacks at every stage. In practice, a sequential pipeline often produces 2 to 4 seconds of response delay, which makes conversation feel unnatural.

A streaming pipeline overlaps the stages. STT streams partial transcripts to the LLM, the LLM streams tokens to TTS, and TTS begins generating audio before the full response is ready. This transforms total latency from roughly VAD + STT + LLM + TTS to something much closer to max(VAD, STT, LLM, TTS). That's the difference between "feels broken" and "feels like a real conversation."

### Cascade vs Speech-to-Speech

There are two architectural options, and most production agents now combine them.

The cascade pipeline (STT to LLM to TTS) is the traditional approach: three separate vendors, three separate models, three separate logs. More moving parts, but you can pick the best model at each layer, redact PII between stages, and swap a vendor without a rewrite. This is what 90% of LiveKit production agents use.

Speech-to-speech (S2S) models skip the text intermediate entirely, passing audio directly through a single model. Lower latency, better prosody preservation, but less debuggable and less mature for tool use.

| Situation | Use |
|---|---|
| Need auditability / transcripts | Cascade |
| Regulated industry (finance, health) | Cascade |
| Tool calling reliability matters | Cascade |
| Sub-300ms latency is critical | S2S |
| Emotional tone / prosody preservation matters | S2S |

### Interruption Handling

One of the trickiest problems in voice pipeline design is barge-in, where the user interrupts the agent mid-response. In a naive pipeline, the agent just keeps talking. In a production pipeline, the system needs to detect the interruption, stop TTS playback immediately, flush any queued audio, and restart the pipeline from STT. LiveKit's framework handles this automatically. When the VAD detects speech while the agent is talking, it fires an interruption event that cancels the active TTS playback and triggers a new STT pass.

---

## Code Structure (Python)

```python
from livekit.agents import AgentSession, Agent, RoomInputOptions
from livekit.plugins import openai, deepgram, cartesia, silero

async def entrypoint(ctx: JobContext):
    session = AgentSession(
        stt=deepgram.STT(),              # speech to text
        llm=openai.LLM(model="gpt-4o"), # language model
        tts=cartesia.TTS(),              # text to speech
        vad=silero.VAD.load(),           # voice activity detection
    )

    await session.start(
        room=ctx.room,
        agent=Agent(instructions="You are a helpful assistant."),
        room_input_options=RoomInputOptions(),
    )
```

The framework provides powerful abstractions for organizing agent behavior, including agent sessions, tasks and task groups, workflows, tools, pipeline nodes, turn detection, agent handoffs, and external data integration.

---

## Integrations

| Layer | Supported Providers |
|---|---|
| STT | Deepgram, AssemblyAI, Google, Azure |
| LLM | OpenAI, Anthropic, Google, Groq, local via Ollama |
| TTS | Cartesia, ElevenLabs, Azure, Google |
| VAD | Silero (default) |

It also integrates with LangChain and LangGraph directly via an LLMAdapter that requires a Pregel-compatible compiled StateGraph.

---

## Deployment

LiveKit provides a cloud-managed solution, LiveKit Cloud, which offers a fully managed, globally distributed infrastructure. Self-hosting runs via Docker or Kubernetes. The server exposes a REST API for room management and uses JWT-based access tokens for participant authentication.

---

## Where It Fits in the AI Stack

If you are building an agentic system that needs a voice interface, LiveKit sits at the transport and orchestration layer. The LLM reasoning, RAG retrieval, and tool use all happen inside the LLM node. LiveKit handles getting audio in and out of that node reliably and at low latency. It is essentially the real-time I/O layer for a voice agent, the same way an HTTP server is the I/O layer for a web API.
