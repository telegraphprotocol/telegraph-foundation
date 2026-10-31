# Telegraph Foundation Model

The shared Foundation Model every Telegraph validator loads once. An Evaluator adapter (the
scoring model of a judgment or hybrid intent) is a small LoRA delta package scored on top of
it, so every validator, executor and adapter author must use exactly these files.

The files are published as assets of a GitHub Release, one release per foundation version.
This repository holds only this README.

## Current release: `modernbert-base-lora16-v1`

Release: <https://github.com/telegraphprotocol/telegraph-foundation/releases/tag/modernbert-base-lora16-v1>

| File | Size | sha256 | Who needs it |
|---|---|---|---|
| `foundation_lora_fp32.onnx` | 599,117,769 B | `0a49b325304bb4cf292e683929689d0457966be264f60bfc94d0072b614bf8e6` | validators, executor operators |
| `foundation_engine.wasm` | 2,686,528 B | `77643d6e1176ab8744801cef93ccb844d0136ed32b8daa865d541f37d13292de` | validators, executor operators |
| `foundation.json` | 1,369 B | `f7de64b999acd3f735118fbe5999abf251fdf188f9797e3a4e36c54ee9e4dbbe` | adapter authors, executor operators |
| `tokenizer.json` | 3,583,486 B | `67bc76f8b9386efba1a761c1c8e5ad439fd0eaf1c2826e9dd1a205c7af4dc456` | validators |
| `SHA256SUMS` | | | everyone |

Adapter authors only need `foundation.json` (the foundation hash, version, tokenizer hash and
LoRA layout) to export and check a package. The ONNX and the engine are needed to score a
package locally and to run a runnable-URL executor.

## Download

```bash
TAG=modernbert-base-lora16-v1
BASE=https://github.com/telegraphprotocol/telegraph-foundation/releases/download/$TAG
mkdir -p foundation && cd foundation
for f in foundation_lora_fp32.onnx foundation_engine.wasm foundation.json tokenizer.json SHA256SUMS; do
  curl -fL --retry 3 -O "$BASE/$f"
done
```

Adapter authors can fetch just the small file:

```bash
curl -fL -o foundation/foundation.json \
  https://github.com/telegraphprotocol/telegraph-foundation/releases/download/modernbert-base-lora16-v1/foundation.json
```

## Verify

Always check the hashes before use. A node refuses to start on a mismatch, and a package
built against a different `foundation.json` is rejected (`foundation_mismatch`).

```bash
sha256sum -c SHA256SUMS          # Linux
shasum -a 256 -c SHA256SUMS      # macOS
```

PowerShell:

```powershell
Get-Content SHA256SUMS | ForEach-Object {
  $h, $f = $_ -split '\s+', 2
  if ((Get-FileHash $f -Algorithm SHA256).Hash.ToLower() -ne $h) { throw "MISMATCH: $f" } else { "OK: $f" }
}
```

## Validator configuration

```bash
FOUNDATION_MODEL_PATH=/path/to/foundation/foundation_lora_fp32.onnx
FOUNDATION_MODEL_HASH=0a49b325304bb4cf292e683929689d0457966be264f60bfc94d0072b614bf8e6
FOUNDATION_MODEL_VERSION=modernbert-base-lora16-v1
FOUNDATION_ENGINE_WASM_PATH=/path/to/foundation/foundation_engine.wasm
FOUNDATION_ENGINE_WASM_HASH=77643d6e1176ab8744801cef93ccb844d0136ed32b8daa865d541f37d13292de
TOKENIZER_JSON=/path/to/foundation/tokenizer.json
```

- Every validator must run the same `FOUNDATION_MODEL_HASH` and engine hash: scores are
  compared bit for bit across validators and executors.
- Both files are hashed before anything loads; a mismatch refuses to start.
- Each foundation worker holds a full copy of the graph: plan about 1.8 GB of RAM per worker.
  `FOUNDATION_POOL_SIZE` sets the number of workers (default: from CPUs and memory, 1 to 8).
- Apply the node's database migrations for the shared foundation before starting the new
  binary.
- On startup the node logs `Foundation Model loaded once …` with the hash and version.

A restart with a different hash or version is a Foundation Model update: adapters built for the
old foundation stop serving and adapters built for the new one are revalidated.

Full guide: [Upgrading to the Shared Foundation Model](https://docs.telegraphprotocol.com/docs/scoring/upgrade-shared-foundation).

## Executor operators

A runnable-URL executor must score packages on these exact files and echo `foundation_hash`
`0a49b325…f8e6` in every reply. The reference executor is `cmd/adapterexec` in the adapter
kit, [telegraph-wasm-baseline](https://github.com/telegraphprotocol/telegraph-wasm-baseline).
A different runtime (for example native ONNX Runtime) gives close but not identical bits and
is rejected.

## Compatibility

| | |
|---|---|
| Base model | [`answerdotai/ModernBERT-base`](https://huggingface.co/answerdotai/ModernBERT-base) @ `8949b909ec900327062f0ebf497f51aef5e6f0c8`, weights unchanged, float32 |
| Graph | ONNX opset 17; 22 layers, hidden size 768. LoRA `A`/`B` for `attn.Wqkv`, `attn.Wo`, `mlp.Wi`, `mlp.Wo` in every layer, `lora.scale`, `head.weight` and `head.bias` are graph inputs |
| Adapter packages | safetensors, `format_version` `1`, `abi` `lora-v1`, `foundation_hash` equal to this release, rank 1 to 16, `max_tokens` up to 512 |
| Engine | WASI reactor (`telegraph_abi() = 2`) built from the kit's `float_runtime/engine` with `build_engine.sh` (oxionnx 0.1.7) |
| Tokenizer | ModernBERT-base `tokenizer.json`, the hash pinned in the node; at most 512 tokens |
| Node | Telegraph nodes with shared Foundation Model support. Full-model `.wasm` adapters are not accepted (`legacy_full_model`) |
| Tools | [telegraph-wasm-baseline](https://github.com/telegraphprotocol/telegraph-wasm-baseline): `export_adapter.py`, `adapterbuild lora`, `adapterexec` |

Built with the kit's `export_foundation.py` (torch 2.12.0, transformers 4.56.0, onnx 1.23.2).
Re-exporting, even from the same base model, can change the bytes and therefore the hash:
always deploy the released files, never a local export.

`foundation.json` records the export check: the split graph matches the base model with the
adapter merged in PyTorch within 2.0e-5 on inputs of 2 to 64 tokens.

## License

The weights and `tokenizer.json` are `answerdotai/ModernBERT-base`'s, unchanged, under its
Apache-2.0 licence.
