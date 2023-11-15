[TOC]

This page exposes steps to make run a coding assistant on your VSCode/VSCodium.

# First try

## Steps
* download a llama2 model
* convert it in gguf format (for llama.cpp execution for CPUs)
* make run llama.cpp as a service
* make run VSCode/VSCodium Continue plugin

## 1 - Prepare your model

The earn time, you should find your prepared model in this place: https://huggingface.co/TheBloke/Llama-2-7B-GGUF.

However, I'll describe how you could apply the conversino by yourself if you need it.

For `llama 2` models, let's go to https://ai.meta.com/llama/ and let's follow the instructions.

Example:

From `/home/user/llm`:

```shell
download.sh
```

After downloading your model, your should have a folder with `consolidated.00.pth` file.
Its parent directory should also contain `tokenizer.model` file
Example:

```
/home/user/llm/models/llama2/tokenizer.model
/home/user/llm/models/llama2/7B/consolidated.00.pth
```

Then, let's convert it to gguf format, to let llama.cpp use it.

From `/home/user/llm`:

```shell
docker run \
  -v ./models/llama2:/models \
  ghcr.io/ggerganov/llama.cpp:full \
  --convert /models/7B
```

Or in a `docker-compose.yml` file:
```yaml
  llama-cpp:
    image: ghcr.io/ggerganov/llama.cpp:full
    volumes:
      - ./models/llama2:/models
    command: --convert /models/7B
```
... And command: `docker compose up llama-cpp`

## 2 - Run llama.cpp service

```shell
docker run \
  -d \
  -p 64256:64256 \
  -v ./models/llama2:/models \
  ghcr.io/ggerganov/llama.cpp:full \
  --server --host 0.0.0.0 --port 64256 -m /models/7B/ggml-model-f16.gguf -c 2048
```

Or in a `docker-compose.yml` file:
```yaml
  llama-cpp:
    image: ghcr.io/ggerganov/llama.cpp:full
    volumes:
      - ./models/llama2:/models
    command: --server --host 0.0.0.0 --port 64256 -m /models/7B/ggml-model-f16.gguf -c 2048
    ports:
      - 64256:64256
```
... And command: `docker compose up -d llama-cpp`

To check your logs:
```shell
docker compose logs -f llama-cpp
```


## 3 - Run Continue plugin

Here are steps to make the link between VSCode/VSCodium Continue plugin and your llama.cpp service

### Install Continue plugin

Could be downloaded from https://marketplace.visualstudio.com/

Direct link I used (install through VSCodium remote ssh, in a linux X64 VM):
`https://marketplace.visualstudio.com/_apis/public/gallery/publishers/Continue/vsextensions/continue/0.7.0/vspackage?targetPlatform=linux-x64`

Then load manually your .vsix file: `Continue.continue-0.7.0@linux-x64.vsix`.

### Setup your Continue plugin to connect to you llama.cpp service

* Server url: `http://172.16.3.63:65432`

* ~/.continue/config.py setup:
```python
from continuedev.libs.llm.llamacpp import LlamaCpp
```
(...)
```python

config = ContinueConfig(
    allow_anonymous_telemetry=False,
    models=Models(
        default=LlamaCpp(
            max_context_length=2048,
            server_url="http://172.16.3.63:64256")
    ),
```
(...)

And make it run with with docker.

Here is a convenient extract of my `docker-compose.yml` file:

```yaml
  continue:
    image: python:3.10.13-bookworm
    working_dir: /continue
    volumes:
      - ./continue/config.py:/root/.continue/config.py
    command:
      - /bin/bash
      - -c
      - |
        pip install continuedev
        python -m continuedev --host 0.0.0.0 --port 65432
    ports:
      - 65432:65432
```

Make it run with: `docker compose up -d continue`

# Conclusion

This expose only the first try.

We know clearly that the chat you'll get won't be powerful, but at least we have a full integration chain.

On next try, we'll discover `rift` full solution. 
