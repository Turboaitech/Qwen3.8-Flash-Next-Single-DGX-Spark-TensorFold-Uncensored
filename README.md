# Qwen3.8-Flash-Next-Single-DGX-Spark-TensorFold-Uncensored

Uncensored-by-default deployment of **Qwen3.8-Flash-Next on one 128 GB NVIDIA DGX Spark**, using TensorFold v1.2.7 and a pinned Keys house-QSA overlay. Scripts and provenance only; model weights download from Hugging Face.

Based on [MiaAI-Lab's recipe](https://github.com/MiaAI-Lab/Qwen3.8-Flash-Next-Single-DGX-Spark-TensorFold) at `b956dba8a468f1cf376d60bbbd9b9ed51ff2f0a9`. Original history, license and credits are retained. See the [upstream README](docs/UPSTREAM_README.md) for engine details and stock-model benchmarks; its defaults differ from this repository.

## Changes

- `ABLIT=1` by default, using the existing upstream overlay loader.
- Default API model ID: `Qwen3.8-Flash-Next-Single-DGX-Spark-TensorFold-Uncensored`.
- The stock checkpoint stays intact. A separate checkpoint view redirects 24 tensors to the overlay: weights and scales of 12 QSA attention output projections.
- GDN outputs, experts, MTP, vision, n-gram embeddings and the chat template retain the pinned NVIDIA weights. This is the pinned **house-QSA** overlay, not a newer QSA+GDN variant.

## Pinned sources

| Component | Source | Revision |
| --- | --- | --- |
| Base | [Qwen/Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next) | NVIDIA export below |
| FP4 QAD checkpoint | [nvidia/Qwen3.8-Flash-Next-FP4-Local](https://huggingface.co/nvidia/Qwen3.8-Flash-Next-FP4-Local) | `b9ce18a6887e18c089fdea8509a33e63429a3300` |
| Keys overlay | [drowzeys/keys-Qwen3.8-Flash-Next-NVFP4-FP4-QAD-house-qsa-L3-47](https://huggingface.co/drowzeys/keys-Qwen3.8-Flash-Next-NVFP4-FP4-QAD-house-qsa-L3-47) | `c44f5b92ee3d307b82aab1c55f4cc4ae1d440344` |
| TensorFold | [ashhart/TensorFold](https://github.com/ashhart/TensorFold) | `93a2e93ecebff8aa52b88e58c5dd222516e69c5f` (v1.2.7, plus upstream patches) |

Overlay file: `fp4-local/house-qsa-oproj-mxfp8-L3-47.safetensors`, about 190 MB. The stock checkpoint is about 96 GiB on disk. Matching overlay shapes and dtypes mean model memory is essentially unchanged.

## Start on a DGX Spark

Accept the overlay's access agreement on Hugging Face using the account associated with your read token. Authenticate locally with `hf auth login`. Never commit tokens.

```bash
git clone https://github.com/Turboaitech/Qwen3.8-Flash-Next-Single-DGX-Spark-TensorFold-Uncensored.git
cd Qwen3.8-Flash-Next-Single-DGX-Spark-TensorFold-Uncensored
./scripts/prepare.sh
./start.sh
```

Stop any existing model server before starting a second copy. For an existing systemd installation, use its service to stop/restart the deployment instead of invoking a competing `start.sh`.

Defaults retain port `8888`, a `262144`-token request window, up to five concurrent requests, FP8 KV, vision and MTP. Memory and request lengths determine actual concurrency. Override the API ID with `SERVED_NAME=my-model ./start.sh`.

```bash
curl -fsS http://127.0.0.1:8888/health
curl -fsS http://127.0.0.1:8888/v1/models
docker inspect -f '{{json .Config.Cmd}}' qwen38-flash-next-tf
python3 tools/toolcheck.py
python3 tools/visioncheck.py
```

The container's checkpoint path should end with `ablit-b9ce18a6-c44f5b92`.

For the stock checkpoint, use `ABLIT=0 ./start.sh restart`; its default API ID returns to `Qwen3.8-Flash-Next`. For systemd, set `Environment=ABLIT=0` in its override and restart through systemd. Configuration precedence is environment, then `.env`, then built-in defaults.

## Validation and limits

On one DGX Spark on 2026-11-20, the view passed all 24 tensor-name/shape/dtype checks and the running container was confirmed to use it. Health and model discovery worked. Tool calling passed 3/3 checks (ordinary, streamed and tool-result continuation). Vision correctly identified a red circle and a blue square. The existing `/v1/messages` endpoint resumed serving requests.

A drafted/undrafted/concurrent consistency run was interrupted while other clients submitted long-context requests; it produced no pass/fail result. No local refusal-rate benchmark, long-context accuracy qualification or controlled speed comparison has been completed for this overlay. Upstream stock-model performance numbers are not measurements of this variant.

`Uncensored` identifies the overlay, not a guarantee of zero refusals or unchanged capabilities. The code retains its original licenses; weights remain subject to their NVIDIA/Qwen licenses and the overlay's access agreement. See [LICENSE](LICENSE), [NOTICE](NOTICE), [CREDITS.md](CREDITS.md) and the linked model cards.
