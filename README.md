# Orpheus TTS
## Overview
Orpheus TTS is an open-source text-to-speech system built on the Llama-3b backbone. Orpheus demonstrates the emergent capabilities of using LLMs for speech synthesis. We offer comparisons of the models below to leading closed models like Eleven Labs and PlayHT in our blog post.

[Check out our blog post](https://canopylabs.ai/model-releases)


https://github.com/user-attachments/assets/ce17dd3a-f866-4e67-86e4-0025e6e87b8a

## Abilities

- **Human-Like Speech**: Natural intonation, emotion, and rhythm that is superior to SOTA closed source models
- **Zero-Shot Voice Cloning**: Clone voices without prior fine-tuning
- **Guided Emotion and Intonation**: Control speech and emotion characteristics with simple tags
- **Low Latency**: ~200ms streaming latency for realtime applications, reducible to ~100ms with input streaming

## Models

We provide three models in this release, and additionally we offer the data processing scripts and sample datasets to make it very straightforward to create your own finetune.

1. [**Finetuned Prod**](https://huggingface.co/canopylabs/orpheus-tts-0.1-finetune-prod) – A finetuned model for everyday TTS applications

2. [**Pretrained**](https://huggingface.co/canopylabs/orpheus-tts-0.1-pretrained) – Our base model trained on 100k+ hours of English speech data


### Inference

#### Simple setup on colab
1. [Colab For Tuned Model](https://colab.research.google.com/drive/1KhXT56UePPUHhqitJNUxq63k-pQomz3N?usp=sharing) (not streaming, see below for realtime streaming) – A finetuned model for everyday TTS applications.
2. [Colab For Pretrained Model](https://colab.research.google.com/drive/10v9MIEbZOr_3V8ZcPAIh8MN7q2LjcstS?usp=sharing) – This notebook is set up for conditioned generation but can be extended to a range of tasks.

#### Streaming Inference Example

1. Clone this repo
   ```bash
   git clone https://github.com/canopyai/Orpheus-TTS.git
   ```
2. Navigate and install packages
   ```bash
   cd Orpheus-TTS && pip install orpheus-speech # uses vllm under the hood for fast inference
   ```
   vllm pushed a slightly buggy version on March 18th so some bugs are being resolved by reverting to `pip install vllm==0.7.3` after `pip install orpheus-speech`
4. Run the example below:
   ```python
   from orpheus_tts import OrpheusModel
   import wave
   import time
   
   model = OrpheusModel(model_name ="canopylabs/orpheus-tts-0.1-finetune-prod")
   prompt = '''Man, the way social media has, um, completely changed how we interact is just wild, right? Like, we're all connected 24/7 but somehow people feel more alone than ever. And don't even get me started on how it's messing with kids' self-esteem and mental health and whatnot.'''

   start_time = time.monotonic()
   syn_tokens = model.generate_speech(
      prompt=prompt,
      voice="tara",
      )

   with wave.open("output.wav", "wb") as wf:
      wf.setnchannels(1)
      wf.setsampwidth(2)
      wf.setframerate(24000)

      total_frames = 0
      chunk_counter = 0
      for audio_chunk in syn_tokens: # output streaming
         chunk_counter += 1
         frame_count = len(audio_chunk) // (wf.getsampwidth() * wf.getnchannels())
         total_frames += frame_count
         wf.writeframes(audio_chunk)
      duration = total_frames / wf.getframerate()

      end_time = time.monotonic()
      print(f"It took {end_time - start_time} seconds to generate {duration:.2f} seconds of audio")
   ```

#### Prompting

1. The `finetune-prod` models: for the primary model, your text prompt is formatted as `{name}: I went to the ...`. The options for name in order of conversational realism (subjective benchmarks) are "tara", "leah", "jess", "leo", "dan", "mia", "zac", "zoe". Our python package does this formatting for you, and the notebook also prepends the appropriate string. You can additionally add the following emotive tags: `<laugh>`, `<chuckle>`, `<sigh>`, `<cough>`, `<sniffle>`, `<groan>`, `<yawn>`, `<gasp>`.

2. The pretrained model: you can either generate speech just conditioned on text, or generate speech conditioned on one or more existing text-speech pairs in the prompt. Since this model hasn't been explicitly trained on the zero-shot voice cloning objective, the more text-speech pairs you pass in the prompt, the more reliably it will generate in the correct voice.

<!-- 3. The research model: the prompt that should get passed to the model has `prompt + " " + "<{emotion}>"` at the end. It should also not have the `{name}:` prefix as it is only trained on one voice. This model is not designed to be used in production. Rather, it's main goal is to show how LLMs can easily support tags to guide controllable emotional generations, and for now will perform worse on other metrics.
 -->

Additionally, use regular LLM generation args like `temperature`, `top_p`, etc. as you expect for a regular LLM. `repetition_penalty>=1.1`is required for stable generations. Increasing `repetition_penalty` and `temperature` makes the model speak faster.


## Finetune Model

Here is an overview of how to finetune your model on any text and speech.
This is a very simple process analogous to tuning an LLM using Trainer and Transformers.

You should start to see high quality results after ~50 examples but for best results, aim for 300 examples/speaker.

1. Your dataset should be a huggingface dataset in [this format](https://huggingface.co/datasets/canopylabs/zac-sample-dataset)
2. We prepare the data using this [this notebook](https://colab.research.google.com/drive/1wg_CPCA-MzsWtsujwy-1Ovhv-tn8Q1nD?usp=sharing). This pushes an intermediate dataset to your Hugging Face account which you can can feed to the training script in finetune/train.py. Preprocessing should take less than 1 minute/thousand rows.
3. Modify the `finetune/config.yaml` file to include your dataset and training properties, and run the training script. You can additionally run any kind of huggingface compatible process like Lora to tune the model.
   ```bash
    pip install transformers datasets wandb trl flash_attn torch
    huggingface-cli login <enter your HF token>
    wandb login <wandb token>
    accelerate launch train.py
   ```
## Also Check out

While we can't verify these implementations are completely accurate/bug free, they have been recommended on a couple of forums, so we include them here:

1. [A lightweight client for running Orpheus TTS locally using LM Studio API](https://github.com/isaiahbjork/orpheus-tts-local) 
2. [Gradio WebUI that runs smoothly on WSL and CUDA](https://github.com/Saganaki22/OrpheusTTS-WebUI)


# Checklist

- [x] Release 3b pretrained model and finetuned models
- [ ] Release pretrained and finetuned models in sizes: 1b, 400m, 150m parameters
- [ ] Fix glitch in realtime streaming package that occasionally skips frames.
- [ ] Fix voice cloning Colab notebook implementation
