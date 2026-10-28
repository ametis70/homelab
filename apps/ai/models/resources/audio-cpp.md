# audio.cpp model services

Audio.cpp models follow the existing KServe model layout. Each model gets its own Flux Kustomization and InferenceService, reusing:

- `components/model-download-template`
- `components/audio-cpp-inference-template`
- `apps/ai/audio-cpp-rocm-runtime`

Models and predictor services:

| Model | InferenceService | OpenAI-compatible API base |
|---|---|---|
| Kokoro TTS | `audio-cpp-kokoro-tts` | `http://audio-cpp-kokoro-tts-predictor.ai.svc.cluster.local/v1` |
| Qwen3-ASR | `audio-cpp-qwen3-asr` | `http://audio-cpp-qwen3-asr-predictor.ai.svc.cluster.local/v1` |

Weights are pinned to revision `874fcb41ececda20bcbc0782d6b4ea7f7d069ea8` and stored separately on `ai-model-cache`.

The existing standalone `audio-cpp` Deployment is set to zero replicas in the desired manifests. The model Kustomizations depend on it so Flux releases its GPU allocation before starting the two predictors. These changes have **not** been applied; the standalone pod is still running until rollout is explicitly authorized. After the KServe predictors are available, update OpenWebUI's Audio settings to the corresponding base URLs above; that setting is stored in OpenWebUI, not in this GitOps tree.
