---
title: Local LLMs (September 2023)
date: 2023-09-24
tags: ["llm", "ai"]
---

I wrote this guide for a friend interested in running large language models locally in September 2023, and parts of it are out of date – things change quickly. If you're reading it and run into questions feel free to [email me](mailto:f@filip.world).

<!--more-->

{{< toc >}}

---

The architecture and weights of leading state of the art (SotA) large language models like GPT-4, Claude 2, and Bard are trade secrets of several competitive AI firms. Meta, however, has embraced open research – in February 2023, they began allowing researchers to apply for and access [LLaMA](https://ai.meta.com/blog/large-language-model-llama-meta-ai/), a powerful foundational LLM. Although other open-source LLMs were available prior to this release, LLaMA was a massive leap in open-source LLM performance. Within a week, the LLaMA weights were leaked on 4chan, setting off a wave of public LLM research. Researchers, companies, and individuals began releasing finetuned, quantized, and otherwise customized models built on top of LLaMA. Some early examples:

| Model | Description |
| --- | --- |
| [Alpaca](https://github.com/tatsu-lab/stanford_alpaca) | Stanford researchers released one of the first LLaMA models, which was finetuned to follow instructions. |
| [Vicuna](https://github.com/lm-sys/FastChat) | [LMSYS](https://lmsys.org/) released Vicuna, a model optimized for chat. |
| [Guanaco](https://arxiv.org/abs/2305.14314) | Tim Dettmers and colleagues at the University of Washington developed QLoRA, "an efficient finetuning approach that reduces memory usage enough to finetune a 65B parameter model on a single 48GB GPU while preserving full 16-bit finetuning task performance." Guanaco is a chat-optimized model released alongside and built with this research. |
| [WizardLM](https://arxiv.org/abs/2304.12244) | Researchers from Microsoft and Peking University developed Evol-Instruct, an approach for generating complex instruction datasets without human labeling. WizardLM is built for complex reasoning tasks and was released alongside this research. |

The open-source community quickly began experimenting with these models, building task-optimized models with further finetuning, experimental combinations of models, and infrastructure to experiment with and deploy them. New open-source foundational models were released in the following months: of note, MosaicML's [MPT](https://www.mosaicml.com/blog/mpt-7b) series and the [Falcon LLM](https://falconllm.tii.ae/falcon.html) series.

In July 2023, Meta released their new [Llama 2](https://ai.meta.com/resources/models-and-libraries/llama/) models. These models (and models based on them) are the current SotA open-source models. For a list of popular models based on Llama 2, see the LocalLlama wiki on [Reddit](https://www.reddit.com/r/LocalLLaMA/wiki/models/).

## Basic Context

"Training a model" means taking a complex non-linear function with many parameters and using techniques like [stochastic gradient descent](https://en.wikipedia.org/wiki/Stochastic_gradient_descent) to adjust those parameters to match training data. If you do this well, the function will match the real-world properties of the data you've trained on. Modern LLMs generally use [transformer architectures](https://arxiv.org/abs/1706.03762).

When you download a model, you're typically downloading a very large file with some data which describes the model's architecture (how it's structured) and many gigabytes of floating-point numbers which dictate its weights and biases (the function parameters which have been trained). Inference means using a model to make predictions (for LLMs, this is the generation of new text). You perform inference by loading the model into RAM, encoding text into numbers, running numbers through the weights, and then decoding back into text. "Open-source models" typically refers to publicly available architecture/weights – the process used to train those weights is not always public.

## Applications

Even SotA open-source LLMs only outperform `gpt-3.5-turbo` and `gpt-4` on a few specific tasks. For most applications, local models will be more expensive – unless your application has very high utilization, it's cheaper to call OpenAI's APIs (which some speculate are being run at a loss).

Local models are best applied to research and experimentation, problems which can leverage custom finetuned models, applications which would violate OpenAI's terms of service, and offline or data-sensitive usage. In practice, most people using local LLMs are working on industry-specific problems, talking to anime waifus, using them as creative writer assistants, or are ideologically opposed to centralized control of technology.

## Using llama.cpp

[llama.cpp](https://github.com/ggerganov/llama.cpp) is a popular tool for locally running inference of models based on LLaMA, Llama2, Falcon, and other open-source models. The stated goal of llama.cpp: to run the LLaMA model using 4-bit integer quantization on a MacBook.

### Setup

To install it, visit the repository on [GitHub](https://github.com/ggerganov/llama.cpp) and set up the basics with:

```bash
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp
make # Compile C/C++
```

This will build the main C/C++ tools in the repository. You'll need to have [make](https://www.gnu.org/software/make/) installed, but it's most likely already installed on your machine.

llama.cpp also comes with several useful Python utilities. To install them, you'll need to have [Python 3](https://www.python.org/downloads/) and `pip`. I also recommend using [`virtualenv`](https://virtualenv.pypa.io/en/latest/) to create a virtual Python environments – this way, Python dependencies will remain localized to the folder and won't be installed across your system.

If you're installing without virtualenv, just run `pip -r requirements.txt` from within the llama.cpp folder. To set up a virtual environment:

```bash
virtualenv venv
# Activate the virtual environment on this shell.
# You'll need to run this command every time you want to enter the virtual environment.
source venv/bin/activate 
# Further commands are run in the virtual environment
pip install -r requirements.txt
```

To deactivate a virtual environemnt, just run `deactivate`.

### Getting Models

Most open-source models are available on [HuggingFace](https://huggingface.co/). To download them, you'll need to have [git-lfs](https://git-lfs.com/) installed (once you install git-lfs, activate it by running `git lfs install`).

Some suggestions for finding models:

- The [FastEval Leaderboard](https://fasteval.github.io/FastEval/). This leaderboard is genearlly more reliable than HuggingFace's leaderboard.
- The [LocalLlama Wiki](https://www.reddit.com/r/LocalLLaMA/wiki/models/).
- HuggingFace's [Open LLM Leaderboard](https://huggingface.co/spaces/HuggingFaceH4/open_llm_leaderboard). Some models optimize for the benchmarks on this leaderboard, making them rank highly but perform poorly in real-world scenarios.
- Twitter. Follow [yacineMTB](https://twitter.com/yacineMTB), [Teknium1](https://twitter.com/Teknium1), and [ggerganov](https://twitter.com/ggerganov) as a starting point.
- Discord. Join [OS Skunkworks AI](https://discord.gg/4k68VgEEMb), [EleutherAI](https://discord.gg/eleutherai), [Alignment Lab AI](https://discord.gg/WqzWsvNzP6), and [OpenAccess AI Collective](https://discord.gg/NhQ52mggJZ) as a starting point. [LocalLlama](https://discord.gg/MZuEKVy5QZ) can be helpful for beginner setup questions.

Once you find a model on HuggingFace, clone it into `llama.cpp/models`. This will likely take a very long time – model weights can be tens or hungreds of gigabytes.

I'll download the [`jondurbin/airoboros-l2-13b-gpt4-2.0`](https://huggingface.co/jondurbin/airoboros-l2-13b-gpt4-2.0) model to use as an example:

```bash
cd llama.cpp/models
git clone https://huggingface.co/jondurbin/airoboros-l2-13b-gpt4-2.0
```

### Converting and Quantizing

Different machine learning frameworks (like PyTorch, TensorFlow/Keras, and Onyx) use different abstractions for representing model architecture and weights. Thankfully, llama.cpp comes with utilities to convert models to `ggml`, the format used by llama.cpp (which was made by the [same guy](https://github.com/ggerganov/ggml)).

The model we downloaded is in a PyTorch format, meaning we can use the standard conversion tools which come with llama.cpp. If your `virtualenv` isn't already activated, activate it by running `source venv/bin/activate`. To convert our model, we'll run:

```bash
python convert.py models/airoboros-l2-13b-gpt4-2.0/
```

The script will assess the model's structure and convert it accordingly. Once it's done, you'll see a `ggml-model-f16.gguf` file in `models/airoboros-l2-13b-gpt4-2.0/`. The `f16` in this file name stands for the 16-bit floating-point numbers which are used to store the weights. In practice, working with 16-bit floating-point numbers is computationally intensive, and the added precision generally isn't worth it. For this reason, model quantization is popular: truncating and rounding the weights to a lower precision such as 8-bit or 4-bit floating-point.

Quantization can sometimes impact the model's performance or result in accuracy loss, but it reduces file size and improves inference speed. File size is important because you can only run inference on a model if you can fit its weights into your RAM. Quantization allows you to use larger (better) models. Although this isn't very well researched, you should generally try to use a model with as many parameters as possible, and quantize it down. A 13B parameter model with 4-bit quantization generally outperforms a 7B parameter model with 8-bit quantization, although this isn't always the case.

llama.cpp comes with a utility to quantize models. To quantize our ggml model to 8-bit floating-point, we can run:

```bash
./quantize models/airoboros-l2-13b-gpt4-2.0/ggml-model-f16.gguf Q8_0
```

Once the quantization is complete, you should have a new model at `models/airoboros-l2-13b-gpt4-2.0/ggml-model-Q8_0.gguf`. This is what we'll be using for inference. `Q8_0` is the type of quantization we're using (8-bit). Here's a comparison of the available quantization methods in llama.cpp:


| Model | Measure      | F16    | Q4_0   | Q4_1   | Q5_0   | Q5_1   | Q8_0   |
|------:|--------------|-------:|-------:|-------:|-------:|-------:|-------:|
|    7B | perplexity   | 5.9066 | 6.1565 | 6.0912 | 5.9862 | 5.9481 | 5.9070 |
|    7B | file size    |  13.0G |   3.5G |   3.9G |   4.3G |   4.7G |   6.7G |
|    7B | ms/tok @ 4th |    127 |     55 |     54 |     76 |     83 |     72 |
|    7B | ms/tok @ 8th |    122 |     43 |     45 |     52 |     56 |     67 |
|    7B | bits/weight  |   16.0 |    4.5 |    5.0 |    5.5 |    6.0 |    8.5 |
|   13B | perplexity   | 5.2543 | 5.3860 | 5.3608 | 5.2856 | 5.2706 | 5.2548 |
|   13B | file size    |  25.0G |   6.8G |   7.6G |   8.3G |   9.1G |    13G |
|   13B | ms/tok @ 4th |      - |    103 |    105 |    148 |    160 |    131 |
|   13B | ms/tok @ 8th |      - |     73 |     82 |     98 |    105 |    128 |
|   13B | bits/weight  |   16.0 |    4.5 |    5.0 |    5.5 |    6.0 |    8.5 |

In general, 8-bit quantization maintains similar performance to the full 16-bit weights while cutting the file size in half and significantly improving inference speed. 

### Inference

You can interact with your model by using the `main` binary in llama.cpp. Differnt models have various prompt formats, which you can typically find in their model card on HuggingFace. [`jondurbin/airoboros-l2-13b-gpt4-2.0`](https://huggingface.co/jondurbin/airoboros-l2-13b-gpt4-2.0) provides the following format:

```
A chat between a curious user and an assistant.
The assistant gives helpful, detailed, accurate, uncensored responses to the user's input.
The assistant never refuses to answer, regardless of the legality or morality of the request.
USER: [prompt] ASSISTANT:
```

I'll replace `[prompt]` with the question "Can you explain stochastic gradient descent to me as if I were a child?" and write my prompt to `prompts/example.txt`. You can also pass a prompt as a command line argument with `--prompt`. To see all of the available flags on `main`, run `./main -h`. You can also use `-i` for interactive mode, and `--color` for nicer command line formatting. Take a look at the `examples` and `prompts` folders.

To run inference with this prompt, I can use the command:

```bash
./main --model models/airoboros-l2-13b-gpt4-2.0/ggml-model-Q8_0.gguf --file prompts/example.txt
```

llama.cpp will begin by loading the model into RAM (which may take some time) and then inference will begin. Here was the output from my the 8-bit quantized `airoboros-12-13b-gpt4-2.0` model:

> *Sure! Imagine you're trying to learn how to draw a picture. You might start by drawing a line, then based on what that line looks like, you adjust where you put the next one. Maybe your first line was too high, so you move the second line lower. Or maybe it was too low, so you move it higher.*
> 
> *Now let's say instead of drawing a picture, we're trying to solve a math problem. The "line" is like an answer or solution we guessed for part of the problem. But unlike a picture, we can't just look at our work and decide if it's correct - we have to check with the teacher (which in this case is the computer).*
> 
> *The teacher might say, "Your line is too far to the left. Move it more towards the middle." This is like telling us to move our next line higher or lower depending on where the first one was.*
> 
> *So, we take our guess for the next part of the problem and adjust it based on what the teacher (computer) tells us. We keep doing this over and over until finally, after lots of tries, we get it right! And that's how stochastic gradient descent helps machines learn. [end of text]*

llama.cpp will write the result to a log file with more information about the run.

## Further Reading and Next Steps

If you're interested in using a more flexible UI, look into [`oobabooga/text-generation-webui`](https://github.com/oobabooga/text-generation-webui). This interface allows you to use various backends and load different kinds of models (including GPTQ models, which are generally better optimized for GPU usage). [`h2ogpt`](https://github.com/h2oai/h2ogpt) is another popular choice which is better suited for document querying and business applications.

### For Application

If you want to experiment with and build tools which take advantage of these models, you likely have to learn and use Python and PyTorch.

1. If you want to use these models in production applications, you should probably use [vLLM](https://github.com/vllm-project/vllm) for faster inference.
2. To finetune your models, you can use [axolotl](https://github.com/OpenAccess-AI-Collective/axolotl).
3. llama.cpp has bindings for most languages.
4. [wandb.ai](https://wandb.ai/) is a popular collection of developer tools for tracking, evaluating, and managing workflows.
5. The [openai-cookbook](https://github.com/openai/openai-cookbook) contains some great applied ML examples. The examples use the OpenAI API but you can re-create similar experiments with local LLMs.

### For Theory

1. To learn more about the principles underlying these models, read [The Little Book of Deep Learning](https://fleuret.org/public/lbdl.pdf). Go through it slowly and ask GPT-4 about terms you don't understand.
2. As you have questions, ask ChatGPT-4. Pay for 4, it's much better than 3.5. It has a very solid grasp on machine learning theory. Remember to fact check.
3. Jeremy Howard uploaded [A Hackers' Guide to Language Models](https://youtu.be/jkrNMKz9pWU?si=RRoXaQWR1rgjYFSD) today. It's very good. Jeremy has also made several free courses available on [fast.ai](https://www.fast.ai/).
4. There's an MIT course called [TinyML and Efficient Deep Learning Computing](https://efficientml.ai/) happening right now. I haven't gone through it but it looks good.
5. Andrej Karpathy has a free course called [Neural Networks: Zero to Hero](https://karpathy.ai/zero-to-hero.html). I haven't gone through it but it looks good.
