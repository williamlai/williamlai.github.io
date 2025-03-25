---
title: use-aws-polly-generate-audio-dataset
date: 2022-12-05 15:18:17
categories:
tags:
- Syntiant
- NDP101
- AI
- AWS
- Polly
---

Gathering a big enough dataset for training an AI model is always a challenge. In this project, I'm gonna use AWS Polly to generate a voice dataset and use this dataset to train a model on Edge Impulse to identify two voices, "on" and "off", and deploy the model to a Syntiant TinyML board.

<!-- more -->

# Ideas

[Amazon Polly](https://docs.aws.amazon.com/polly/latest/dg/what-is.html) is a cloud service that converts text into spoken audio. Its Text-to-Speech (TTS) service uses deep learning to synthesize natural sounding human speech.

Polly supports many languages, according to [this list](https://docs.aws.amazon.com/polly/latest/dg/voicelist.html). I use several English voices and their variants to generate a base voice dataset. Then I use FFmpeg to create more audio data by changing its speed and pitch.

To be more specific, I'll use Polly to synthesize voices of two words, "on" and "off," with 13 English voices. For each voice, I'll create new voices by changing its speed from 85% to 115% with 2% intervals. For each speed, I'll also change its pitch from 85% to 115% with 2% intervals. That'll give us 13 * 15 * 15 = 2925 samples for each word.

I also need an open set to identify words that are not "on" or "off". To do so, I use Polly to synthesize common words and change their speed and pitch with bigger intervals so that the open set won't relatively too large than the original dataset.

# Generate dataset

### Create one synthesize voice from Polly

There are several Polly SDKs and several Python and Java sample codes. I refer to the [SynthesizeSpeech sample code](https://docs.aws.amazon.com/polly/latest/dg/SynthesizeSpeechSamplePython.html) to create audio.

### Change audio speed and pitch by FFmpeg

The following command can slow down audio at 85% speed.

```bash
ffmpeg -i input_audio.wav -filter:a atempo=0.85 -vn output_audio.wav
```

The following command can increase the pitch of audio by 115%.

```bash
ffmpeg -i input_audio.wav -af asetrate=16000*1.15,aresample=16000 output_audio.wav
```

To make each audio sample the same length for Edge Impulse to create features, I add silence to the beginning and the end.

The following command adds 0.1s silence to the beginning of the audio.

```
ffmpeg -i input_audio.wav -af areverse,apad=pad_dur=0.1s,areverse output_audio.wav
```

The following command adds 0.2s silence to the end of the audio.

```
ffmpeg -i input_audio.wav -af apad=pad_dur=0.2s output_audio.wav
```

### Audio filename

Edge Impulse supports several ways to import an existing dataset. Parsing the filename is one of the solutions. The characters before the first dot will be the label of an audio file. So the filename is like the follows.

* "`on.extra_description.wav`": Label `on`
* "`off.extra_description.wav`": Label `off`
* "`z_openset.extra_description.wav`": Lable `z_openset`

### Use sample code to generate the aduio dataset 

With the above sample code and sample commands, I put the code in the git repository ["generate-audio-dataset"](https://github.com/williamlai/generate-audio-dataset).

Use the following command to generate an audio dataset of "on", "off", and "z_openset" in the default folder output.

```bash
python generate_audio_dataset.py \
    -w 'on' 'off' \
    -v 'Amy' 'Emma' 'Brian' 'Ivy' 'Joanna' 'Kendra' 'Kimberly' 'Salli' 'Joey' 'Justin' 'Matthew' 'Nicole' 'Russell'
```

# Test on Syntiant TinyML

Now we have an audio dataset that contains 10K+ files.

Create a project named "`on-off-switch`".

{% asset_img create_edge_impulse_project.png %}

Go to "Data acquisition", and "Upload data", then upload all files.

{% asset_img upload_files.png %}

Create impulse design with the following settings.

{% asset_img impulse_design.png %}

In the Syntiant processing block, keep the default settings and "Save parameters".

{% asset_img syntiant_processing_block.png %}

In the classifier learning block, change the number of training cycles to 10, add low noise in the data augmentation, then start training.

{% asset_img classifier_learning_block.png %}

As training is complete, go to "Deployment", select "Build firmware", choose Syntiant TinyML, set up "Find posterior parameters," and select "on" and "off" labels.

{% asset_img find_posterior_parameters.png %}

Then select "Build".

{% asset_img build_firmware.png %}

When the build is complete, there will be an archived file with a script for uploading firmware to NDP101 and SAMD21. Run the script with TinyML connected to the computer.

Then we can check the log from UART.

{% asset_img check_uart_log.jpg %}

Now we can test it by saying "on" or "off" we'll see the following logs.

```
Predictions:

    off:        0

    on:         1

    z_openset:  0

Battery = 99%

Predictions:

    off:        1

    on:         0

    z_openset:  0

Battery = 99%
```

# Conclusion

There are several human speech generator available including AWS Polly, which means it'll do if I use other generator. But AWS provide a well-maintained SDK and make things easier, so I use it in this project.

Use synthesized audio to train an audio model looks like a idea of GAN, so I did some search about it, but the goal is slightly different. The goal of GAN is to make audio sounds like human voice, but in this article we just want to do the classifier work. Although I sure use some ideas of GAN would help too, and I hope there will be some tools like this so that SDE or AI engineer can generate a desired dataset in a short time.

### 05/30/2023 updated

This article was completed by the end of 2022, but it is only being published today. One reason for the delay is uncertainty about whether the idea is complete, and another reason is being too busy.

I'm glad that I finally decided to publish it. I hope that in the future, I will continue self-learning.
