To build a **conversational AI agent with voice integration** in 2026, you have excellent open-source options that let you create a natural, real-time voice assistant (listen → understand → reason → speak back). You can run everything **locally/offline** for privacy (no cloud APIs needed), or mix in cloud services for higher quality if desired.

The current best paths fall into two categories:

- **Quick & production-like**: Use a dedicated open-source voice agent framework (handles real-time streaming, interruptions, turn-taking, STT/LLM/TTS orchestration).
- **Simple & fully customizable**: Build a minimal pipeline yourself (great for learning or tight integration with your existing agent logic).

### 1. Recommended Modern Approach (2026 State-of-the-Art)
Use one of these open-source frameworks — they dominate tutorials and production voice agents right now:

| Framework       | Why Choose It in 2026                          | Best For                              | Local/Offline Friendly | Latency | GitHub / Quick Start |
|-----------------|------------------------------------------------|---------------------------------------|------------------------|---------|----------------------|
| **LiveKit Agents** | Full real-time WebRTC infra + agent framework. Low-latency, interruptions, multimodal possible. Used by big players. | Scalable, phone-like voice agents, enterprise feel | Yes (self-host)       | Ultra-low (~200–500 ms) | github.com/livekit/agents → docs.livekit.io/agents |
| **Pipecat**     | Flexible Python pipelines, very composable. No vendor lock-in. Great docs & community. | Custom logic, rapid prototyping, multimodal | Fully local possible  | Low     | github.com/pipecat-ai/pipecat → pipecat.ai/docs |
| **TEN Framework** | Multimodal focus, extensions for STT/LLM/TTS. | Real-time assistants with vision/audio | Yes                   | Low     | github.com/TEN-framework/ten-framework |

**Fastest way to get started (Pipecat – most beginner-friendly in 2026)**

1. Install:
   ```bash
   pip install "pipecat-ai[local]"  # or full: pipecat-ai[all]
   ```

2. Minimal example (100% local/offline – uses Whisper.cpp or Faster-Whisper for STT, Ollama for LLM, Piper/XTTS for TTS):
   ```python
   # voice_agent.py
   import asyncio
   from pipecat.frames.frames import LLMMessagesFrame
   from pipecat.pipeline.pipeline import Pipeline
   from pipecat.pipeline.runner import PipelineRunner
   from pipecat.pipeline.task import PipelineTask
   from pipecat.services.openai import OpenAILLMContextAggregator  # swap for Ollama
   from pipecat.services.ollama import OllamaLLMService
   from pipecat.services.whisper import WhisperSTTService  # or faster-whisper
   from pipecat.services.piper import PiperTTSService
   from pipecat.transports.webrtc import WebRTCTransport  # or daily/pipecat transport

   async def main():
       stt = WhisperSTTService()                   # Local STT
       llm = OllamaLLMService(model="llama3.1")    # Your local LLM
       tts = PiperTTSService(voice="en_US-lessac-medium")  # Fast local TTS

       # Simple pipeline: mic → STT → LLM → TTS → speaker
       pipeline = Pipeline([stt, llm, tts])  # Add context aggregator if needed

       task = PipelineTask(pipeline)
       runner = PipelineRunner()
       await runner.run(task)

   if __name__ == "__main__":
       asyncio.run(main())
   ```

   Run: `python voice_agent.py`  
   Speak → it listens (VAD detects pauses), transcribes, reasons with Ollama, speaks back.

   Customize:
   - Add your persona/memory (e.g., load from Mem0/Neo4j before LLM call).
   - Use XTTS-v2 for voice cloning: swap Piper → CoquiTTSService.
   - Add interruptions: Pipecat/LiveKit handle VAD + barge-in natively.

**Alternative quick path**: LiveKit Agents (if you want WebRTC/phone integration later)  
Follow their 20-min tutorial: docs.livekit.io/agents/quickstarts/python → swap in Ollama + local STT/TTS.

### 2. Minimal DIY Pipeline (19–50 lines, fully local)
If you want ultimate control or to integrate with your federation agent:

```python
# super simple local voice loop
import asyncio
import sounddevice as sd
import numpy as np
from faster_whisper import WhisperModel          # pip install faster-whisper
from ollama import AsyncClient                   # pip install ollama
from TTS.api import TTS                          # pip install TTS (Coqui)

whisper_model = WhisperModel("base.en", device="cpu")  # or "cuda" if GPU
tts = TTS("tts_models/multilingual/multi-dataset/xtts_v2").to("cpu")  # or GPU
ollama_client = AsyncClient()

async def listen():
    print("Listening...")
    audio = sd.rec(int(8 * 16000), samplerate=16000, channels=1, dtype='float32')
    sd.wait()
    segments, _ = whisper_model.transcribe(audio.flatten(), language="en")
    return " ".join([s.text for s in segments]).strip()

async def think(text):
    response = await ollama_client.chat(
        model='llama3.1',
        messages=[{'role': 'user', 'content': text}]
    )
    return response['message']['content']

def speak(text):
    wav = tts.tts(text=text, speaker_wav=None)  # add speaker_wav for cloning
    sd.play(wav, samplerate=tts.synthesizer.output_sample_rate)
    sd.wait()

async def main_loop():
    while True:
        user_text = await listen()
        if user_text.lower() in ["exit", "quit", "stop"]:
            print("Goodbye!")
            break
        if user_text:
            print(f"You: {user_text}")
            ai_text = await think(user_text)
            print(f"AI: {ai_text}")
            speak(ai_text)

asyncio.run(main_loop())
```

Enhancements:
- Add VAD (Voice Activity Detection) → use silero-vad or webrtcvad to stop recording on silence.
- Streaming → use Pipecat/LiveKit for real-time partial responses.
- Custom persona → Inject system prompt: "You are a witty federation agent named Klasko..."

### Quick Local Stack Recommendation (Privacy + Quality)
- **STT**: Faster-Whisper (or Whisper.cpp for CPU-only)
- **LLM**: Ollama (Llama 3.1 8B/70B, or Gemma-2-27B)
- **TTS**: Piper (fastest CPU) or XTTS-v2 (best quality + cloning)
- **Orchestration**: Pipecat (easiest full-featured) or raw Python

### Tips for Your Setup
- **Hardware**: 16+ GB RAM + decent GPU (RTX 3060+) for smooth real-time. CPU-only works but slower.
- **Latency goal**: 500–1500 ms end-to-end (possible locally).
- **Voice customization**: XTTS-v2 clones from 3–10s sample — record your voice for a personal agent sound.
- **Integration**: Hook into your federation memory (Mem0/Neo4j) by querying before/after LLM call.

Start with Pipecat + local models — you can have a talking agent in under 30 minutes. If you hit issues (e.g., OS-specific install, want GStreamer LAN transport, or federation persona injection), share your OS/hardware and I can give exact commands/code tweaks!