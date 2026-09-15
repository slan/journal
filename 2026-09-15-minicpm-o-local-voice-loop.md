# MiniCPM-o 4.5 local voice loop

date: 2026-09-15 16:57
mode: voice
tags: [voice, llm, minicpm-o, local-inference, docker, tool-calling, rtx-4090]

## Context
Voice session thinking through running a local full-duplex speech model on the home 4090. Starting point was MiniCPM-o 4.5 (OpenBMB, ~9B, end-to-end on SigLip2 + Whisper-medium + CosyVoice2 + Qwen3-8B), which does real-time full-duplex omni-modal streaming — see, listen and speak simultaneously without blocking. Goal is to compare it against ChatGPT voice mode and work out what architecture actually gets a usable local voice assistant.

## Ideas

### cascaded-asr-llm-tts
Use MiniCPM-o purely as speech-to-text, route the transcript to a more powerful text model, then send the reply back out through TTS. Rejected in session: this collapses back into a turn-based pipeline and throws away the duplex behaviour and prosody, which is the main reason to run this model at all. Status: abandoned.

### small-model-owns-loop-delegates-via-tools
Middle ground. MiniCPM-o owns the conversational loop — turn-taking, interruption handling, backchannels, filler speech — and delegates anything needing real reasoning to a bigger model exposed as a tool call. Keeps duplex latency while getting better answers. Open seam: what the small model says during the 2–20s the big model is thinking, and whether it can hold a coherent thread when the answer lands. Status: exploring.

### dockerised-three-container-stack
Docker on Windows, WSL2 backend for GPU passthrough. Container one: model server (vLLM or the official MiniCPM-o demo image) on the 4090. Container two: thin orchestration loop owning the websocket for mic-in / audio-out. Container three: tool-call target behind a plain HTTP endpoint, so local Qwen and a remote model are swappable without touching the voice loop. Phase one is just the demo image plus a headset; phase two adds the orchestration container. Status: seed.

## Decisions
- Architecture is the middle ground: small model owns the conversation loop, delegates hard reasoning to a bigger model via tool calls.
- Drop the cascaded ASR → big model → TTS design.
- Everything runs in Docker on Windows rather than bare WSL.
- Two evaluation tracks, in order: (1) bare model — thinking capability and interaction/interruption quality, (2) reliability of tool calls emitted mid-speech.
- Implementation to be scaffolded by Codex or Claude Code, not designed further in voice.

## Open questions
- Is the 8B brain actually the bottleneck for real use, or do latency and interruption handling matter more?
- What does the small model say while the big model is thinking, and can it do that filler naturally?
- Can it emit reliable tool calls mid-speech at all?
- How does it actually compare to ChatGPT voice mode in practice?

## Next actions
- [ ] Pull the MiniCPM-o 4.5 demo image and run bare model on the 4090 with a headset.
- [ ] Evaluate bare-model thinking quality and interaction/interruption behaviour.
- [ ] Attempt to trigger tool calls mid-conversation.
- [ ] CC: scaffold the docker-compose stack — model server, orchestration loop with websocket mic-in/audio-out, swappable HTTP reasoning endpoint.

## Quotes
- "Couldn't there be a middle ground — this model can be tool calling, so it could just talk to me while invoking a tool in the background that would be handled by a more powerful agent."
- "I'm using Docker on Windows mostly. I can use WSL as well, but I would prefer everything to be in Docker."
