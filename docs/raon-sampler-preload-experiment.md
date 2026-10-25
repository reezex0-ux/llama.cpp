# Experimental loader-prepared backend sampler retention (not adopted)

**Status: experimental, not recommended for production. Not included in KRAFTON PR #1.**

This branch extends the [validated Raon ROCm optimizations PR](https://github.com/krafton-ai/llama.cpp/pull/1) with a separate attempt to preserve the backend sampler already prepared by the model loader through the server slot’s initial reset. When request parameters differ, the code falls back to a fresh registration. After a model unload and reload, a **newly** loader-prepared sampler is adopted; no executable HIP graph is persisted across model lifetimes.

## M8 results (2026-10-26 KST)

Device: RX 7900 XTX, ROCm 7.2.1. Model: Raon-1 mini preview IQ4_XS, ctx 8192, one slot, batch 2048/ubatch 512, backend sampling and previous-request reuse on. The only tested toggle within each matched pair was loader preload on/off. Real server defaults used fixed seed 1234 for reproducing the test. Prompt caching and streaming were disabled. Model/server processes were freshly launched, but OS/file caches were not cleared. Diagnostic tracing was disabled for timings. Each thinking mode used **two matched fresh-process pairs**, with an initial request and an immediately repeated request.

| Metric (mean) | Preload OFF | Preload ON | Difference |
| --- | ---: | ---: | ---: |
| Thinking OFF, first HTTP | 4.584 s | 4.446 s | −0.138 s |
| Thinking OFF, repeat HTTP | 2.182 s | 2.254 s | +0.072 s |
| Thinking OFF, launch through first response | 28.370 s | 35.327 s | +6.957 s |
| Thinking ON, first HTTP | 7.454 s | 7.520 s | +0.066 s |
| Thinking ON, repeat HTTP | 5.375 s | 5.539 s | +0.164 s |
| Thinking ON, launch through first response | 45.262 s | 45.223 s | −0.039 s |

The nominal first-HTTP improvement for thinking OFF did **not** repeat: individual paired differences were −298 ms and +22 ms. The thinking ON first-HTTP differences were +108 ms and +25 ms. Repeated HTTP completed about 3% slower on average with preload enabled. The startup/ready interval varied substantially among trials and cannot be attributed to this patch from these data. **No consistent improvement** was established, so the option was not enabled in the production profile.

Correctness was verified with 36 diagnostic HTTP requests (18 matched comparisons) in addition to 16 performance requests (8 matched comparisons): all **52 requests / 26 paired comparisons** preserved content/reasoning output, token counts and finish reasons, with **440 exact matching** reported log-probability/probability values. Trace verified first default-load and in-process sleep/unload/reload preload hits, initial-parameter mismatch misses, and normal model/context teardown. All 12 test wrappers exited cleanly. The tests do not prove broad model, architecture, or multi-slot correctness.

The existing loader warmup uses only 1–2 tokens; first actual chat prefill may still require shape-specific graph preparation and other backend first-use work. Identifying the remaining cold-path cost requires separated instrumentation. Do not claim that this branch removes model startup latency or preserves GPU executable graphs across unload/reload.

The recommended upstream contribution is [PR #1](https://github.com/krafton-ai/llama.cpp/pull/1), which **does not include** this experiment.
