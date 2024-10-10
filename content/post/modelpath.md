---
title: Use $MODELPATH
date: 2024-05-21
tags: ["machine learning"]
---

## The Problem

On-device AI models are becoming more common, but by default, applications store them in different places.

<!--more-->

| Application | Default | How to Change |
| --- | --- | --- |
| [HuggingFace Transformers](https://huggingface.co/docs/transformers/main/en/index) | `~/.cache/huggingface/hub` | Set `TRANSFORMERS_CACHE` |
| [LM Studio](https://lmstudio.ai) | `~/.cache/lm-studio/models` | My Models -> Change |
| [Ollama](https://ollama.com) | `~/.ollama` | Set `OLLAMA_MODELS` |
| [Jan](https://jan.ai) | `~/jan/models` | Advanced Settings -> Jan Data Folder |
| [GPT4All](https://gpt4all.io/index.html) | `~/.cache/gpt4all/` | Unclear |

These applications (and new ones which will come) are becoming popular with non-technical audiences. A user who tries one of these applications is more likely to try another. When they do, they'll be confused – their models are missing!

Unless they realize that they have to change the default models directory, the friction of downloading 10+ GB of weights comes *before they can try the application*. At this point, many users—especially those with slow internet connections—will give up.

Technical users are likely to already have models on their device. Although they can figure something like this out, it adds friction.

## The Solution

By agreeing to save AI models to a standard path, developers could simplify things for their users, library developers, and providers like HuggingFace.

My suggestion:

Applications which run AI models on-device should save them within `$MODELPATH`. `$MODELPATH` should default to `~/.models` on Unix and `%USERPROFILE%\.models` on Windows.

Aside from new model downloads, `$MODELPATH` should be read-only. Per-model config files should be stored somewhere else.

Models should be saved to `$MODELPATH/<provider-url>/<author>/<model>/`. For example – if a user installs [`meta-llama/Meta-Llama-3-8B`](https://huggingface.co/meta-llama/Meta-Llama-3-8B) from HuggingFace, the model should be saved to `$MODELPATH/huggingface.co/meta-llama/Meta-Llama-3-8B/`.
