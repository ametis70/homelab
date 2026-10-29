# audio.cpp model services

Audio.cpp models follow the existing KServe model layout. Each model has one Flux Kustomization and one InferenceService. The predictor pod runs a model-download init container before the audio.cpp inference container. The init container downloads the pinned GGUF into `ai-model-cache`, verifies SHA-256, and skips the download on later pod starts when the cached file is valid. Both models reuse `components/audio-cpp-inference-template` and the shared runtime at `apps/ai/audio-cpp-rocm-runtime`.

| Model | InferenceService | OpenAI-compatible API base |
|---|---|---|
| Kokoro TTS | `audio-cpp-kokoro-tts` | `http://audio-cpp-kokoro-tts-predictor.ai.svc.cluster.local/v1` |
| Qwen3-ASR | `audio-cpp-qwen3-asr` | `http://audio-cpp-qwen3-asr-predictor.ai.svc.cluster.local/v1` |

Weights are pinned to revision `874fcb41ececda20bcbc0782d6b4ea7f7d069ea8` and stored in revision-specific directories on `ai-model-cache`. The revision hash is used for cache reproducibility, not in the Flux Kustomization or InferenceService names. KServe Deployment pods still have the normal ReplicaSet hash and generated pod suffix.

The standalone `audio-cpp` Deployment is set to zero replicas in the desired manifests. Model Kustomizations depend on it so Flux releases its GPU allocation before starting the predictors. OpenWebUI's Audio settings are stored in its database; set their base URLs to the predictor addresses above after the predictors are ready.
