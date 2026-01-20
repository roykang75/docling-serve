<p align="center">
  <a href="https://github.com/docling-project/docling-serve">
    <img loading="lazy" alt="Docling" src="https://github.com/docling-project/docling-serve/raw/main/docs/assets/docling-serve-pic.png" width="30%"/>
  </a>
</p>

# Docling Serve

Running [Docling](https://github.com/docling-project/docling) as an API service.

📚 [Docling Serve documentation](./docs/README.md)

- Learning how to [configure the webserver](./docs/configuration.md)
- Get to know all [runtime options](./docs/usage.md) of the API
- Explore useful [deployment examples](./docs/deployment.md)
- And more

> [!NOTE]
> **Migration to the `v1` API.** Docling Serve now has a stable v1 API. Read more on the [migration to v1](./docs/v1_migration.md).

## Getting started

Install the `docling-serve` package and run the server.

```bash
# Using the python package
pip install "docling-serve[ui]"
docling-serve run --enable-ui

# Using container images, e.g. with Podman
podman run -p 5001:5001 -e DOCLING_SERVE_ENABLE_UI=1 quay.io/docling-project/docling-serve
```

The server is available at

- API <http://127.0.0.1:5001>
- API documentation <http://127.0.0.1:5001/docs>
- UI playground <http://127.0.0.1:5001/ui>

![API documentation](img/fastapi-ui.png)

Try it out with a simple conversion:

```bash
curl -X 'POST' \
  'http://localhost:5001/v1/convert/source' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
    "sources": [{"kind": "http", "url": "https://arxiv.org/pdf/2501.17887"}]
  }'
```

### Container Images

The following container images are available for running **Docling Serve** with different hardware and PyTorch configurations:

#### 📦 Distributed Images

| Image | Description | Architectures | Size |
|-------|-------------|----------------|------|
| [`ghcr.io/docling-project/docling-serve`](https://github.com/docling-project/docling-serve/pkgs/container/docling-serve) <br> [`quay.io/docling-project/docling-serve`](https://quay.io/repository/docling-project/docling-serve) | Base image with all packages installed from the official PyPI index. | `linux/amd64`, `linux/arm64` | 4.4 GB (arm64) <br> 8.7 GB (amd64) |
| [`ghcr.io/docling-project/docling-serve-cpu`](https://github.com/docling-project/docling-serve/pkgs/container/docling-serve-cpu) <br> [`quay.io/docling-project/docling-serve-cpu`](https://quay.io/repository/docling-project/docling-serve-cpu) | CPU-only variant, using `torch` from the PyTorch CPU index. | `linux/amd64`, `linux/arm64` | 4.4 GB |
| [`ghcr.io/docling-project/docling-serve-cu126`](https://github.com/docling-project/docling-serve/pkgs/container/docling-serve-cu126) <br> [`quay.io/docling-project/docling-serve-cu126`](https://quay.io/repository/docling-project/docling-serve-cu126) | CUDA 12.6 build with `torch` from the cu126 index. | `linux/amd64` | 10.0 GB |
| [`ghcr.io/docling-project/docling-serve-cu128`](https://github.com/docling-project/docling-serve/pkgs/container/docling-serve-cu128) <br> [`quay.io/docling-project/docling-serve-cu128`](https://quay.io/repository/docling-project/docling-serve-cu128) | CUDA 12.8 build with `torch` from the cu128 index. | `linux/amd64` | 11.4 GB |

#### 🚫 Not Distributed

An image for AMD ROCm 6.3 (`docling-serve-rocm`) is supported but **not published** due to its large size.

To build it locally:

```bash
git clone --branch main git@github.com:docling-project/docling-serve.git
cd docling-serve/
make docling-serve-rocm-image
```

For deployment using Docker Compose, see [docs/deployment.md](docs/deployment.md).

Coming soon: `docling-serve-slim` images will reduce the size by skipping the model weights download.


## GPU 가속 설정 가이드 (한국어)

Docling Serve에서 GPU를 사용하여 OCR 및 문서 처리 속도를 높이려면 다음 단계를 따르세요.

### 1. CUDA 지원 PyTorch 설치

기본 설치 시 CPU 전용 PyTorch가 설치될 수 있습니다. GPU를 사용하려면 CUDA 버전 PyTorch를 설치해야 합니다.

```bash
# 가상환경 활성화 (Windows)
.\venv-docling\Scripts\Activate.ps1

# 기존 CPU 버전 제거 후 CUDA 버전 설치
pip uninstall torch torchvision torchaudio -y
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu126
```

> [!NOTE]
> CUDA 버전에 맞게 선택하세요:
> - CUDA 11.8: `cu118`
> - CUDA 12.1: `cu121`
> - CUDA 12.4: `cu124`
> - CUDA 12.6: `cu126`

### 2. CUDA 설치 확인

```bash
python -c "import torch; print('CUDA available:', torch.cuda.is_available()); print('PyTorch version:', torch.__version__)"
```

정상 출력 예시:
```
CUDA available: True
PyTorch version: 2.9.1+cu126
```

### 3. GPU 디바이스 설정

환경변수를 통해 GPU 사용을 활성화합니다:

```powershell
# Windows PowerShell
$env:DOCLING_DEVICE = "cuda"
docling-serve run --enable-ui
```

```bash
# Linux/macOS
export DOCLING_DEVICE=cuda
docling-serve run --enable-ui
```

### 4. RapidOCR GPU 사용 시 주의사항

RapidOCR의 GPU 지원은 다음 조건이 필요합니다:

1. **CUDA PyTorch 설치** (위 단계 완료)
2. **torch 백엔드 사용**: RapidOCR는 기본적으로 `onnxruntime` 백엔드를 사용합니다. `torch` 백엔드 사용 시 GPU가 활성화됩니다.

정상적으로 GPU가 활성화되면 로그에서 다음과 같이 표시됩니다:
```
[RapidOCR] device_config.py:XX: Using CUDA device
```

CPU로 폴백되는 경우:
```
WARNING:docling.utils.accelerator_utils:CUDA is not available in the system. Fall back to 'CPU'
[RapidOCR] device_config.py:50: Using CPU device
```

> [!TIP]
> GPU 메모리가 부족한 경우 (예: RTX 2060 6GB), OCR은 CPU로 두고 Layout/Table 모델만 GPU를 사용하는 것이 더 안정적일 수 있습니다.

---

### Demonstration UI

An easy to use UI is available at the `/ui` endpoint.

![Input controllers in the UI](img/ui-input.png)

![Output visualization in the UI](img/ui-output.png)

## Get help and support

Please feel free to connect with us using the [discussion section](https://github.com/docling-project/docling/discussions).

## Contributing

Please read [Contributing to Docling Serve](https://github.com/docling-project/docling-serve/blob/main/CONTRIBUTING.md) for details.

## References

If you use Docling in your projects, please consider citing the following:

```bib
@techreport{Docling,
  author = {Docling Contributors},
  month = {1},
  title = {Docling: An Efficient Open-Source Toolkit for AI-driven Document Conversion},
  url = {https://arxiv.org/abs/2501.17887},
  eprint = {2501.17887},
  doi = {10.48550/arXiv.2501.17887},
  version = {2.0.0},
  year = {2025}
}
```

## License

The Docling Serve codebase is under MIT license.

## IBM ❤️ Open Source AI

Docling has been brought to you by IBM.
