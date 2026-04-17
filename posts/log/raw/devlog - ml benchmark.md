---
title: Turning the mania benchmark into actual ML
slug: turning-the-mania-benchmark-into-actual-ml
type: experiment
status: draft
date: undated
updated: 2026-04-17
tags:
  - ml
  - benchmark
  - audio
  - osu
  - sequence-modeling
summary: The project moved from API-driven LLM experiments to an audio-first ML benchmark with a large beatmap dataset and a leakage-free training pipeline.
verification_status: partial
verified_on:
---

# Turning the mania benchmark into actual ML

## Context

This started out as a total API nightmare. I had a pipeline that grabbed the `.osz` file, pulled out the notes, turned them into frames, batched them, sent them to an LLM API, converted the output to a replay, scored it, and then stared at the result.

None of those models got above roughly 2% accuracy. That made sense. They were not built for timing or for predicting event sequences that actually matter.

That was the point where the project stopped being “LLMs play osu” and became a real ML benchmark instead.

## Goal

- build a legitimate machine-learning benchmark instead of an API gimmick
- train a model from scratch and measure it honestly
- predict timed events from audio rather than from leaked chart features

## Environment

- osu! / mania beatmap data packaged as `.osz`
- dataset downloads from song packs and API-assisted metadata collection
- TCN model as the first real baseline
- audio features built with `librosa`
- replay generation available but not yet fully looped into training

## The Actual Problem

The project kept hitting three different classes of failure:

- the API-based LLM approach was structurally wrong for rhythm timing
- the first “real” ML pipeline leaked labels into the features
- scaling the dataset and training loop introduced operational pain that had to be solved before iteration could become real

## What Changed

### Dataset scaling nightmare

I needed way more data to make this work. I first looked at Etterna packs, but the download speed cap was around 1 MB/s and the torrents were barely seeded.

So I pivoted again. I built an osu API v2 client to grab beatmap metadata. The API did not allow direct downloads, so I worked around it with cookie auth for direct grabs. That was unreliable, but good enough for a couple of runs.

Then I remembered osu song packs had much better download speed at around 50 MB/s. Instead of scraping one by one, I hit the packs endpoint and automated the whole thing.

I wrote a script to:

- list all the packs
- fetch beatmapsets and show links through osu API v2
- download the zip files
- dump everything into a data folder
- dedupe by beatmapset ID
- keep track of unique songs

That produced 336 packs, around 6.6k beatmapsets, 5.8k unique songs, and about 51 GB of raw data.

### My first ML try

I split the dataset by maps instead of frames to avoid leakage.

Then I trained a tiny TCN model. Press F1 score came back as 1.0, which looked fake because it was fake.

The features for lanes 0-3 were literally the same as the press labels. Same frame and same positions. The model was copying the input to the output.

Releases were worse because the features marked holds as `1` the whole time, while the labels only marked `1` on the exact release frame. So the model cheated on presses and fell apart on releases.

### Audio rewrite

I decided the model had to predict from sound, not from the chart.

So I redid the feature pipeline:

- pull audio from `.osz` using the per-difficulty `AudioFilename`
- load with `librosa` at 22050 Hz
- use 10 ms hop length
- build an 80-bin mel spectrogram
- derive onset strength and RMS
- aggregate to a 20 ms grid to match labels
- save features as `float16`
- normalize based on train split stats only

Features ended up at 82 pooled dims or 164 stacked dims. No note features were kept.

### Operational constraints

Training on the full dataset took roughly 10 hours.

I dealt with that by adding:

- checkpoints for model and optimizer
- resume support
- tiny validation runs for faster sanity checks
- the option of using a cloud GPU later if needed

## Verification

### Early audio results on 200 maps

Training felt much more legitimate after the rewrite.

- press F1 around 0.35
- release F1 around 0.06 to 0.09
- hold F1 around 0.25

With a +/-1 frame tolerance:

- presses up to 0.42
- releases around 0.09
- holds around 0.17

That was not impressive, but it was honest. The model was clearly picking up onsets better than releases.

### Eval wake-up call

Micro-F1 hovered around 0.25 to 0.33 depending on thresholds. Precision was mediocre, recall was better, releases dragged the metric down, and holds were unstable.

The copy-last baseline also looked weirdly strong under tolerant evaluation, which was another reminder that frame classification is not the same thing as actually playing mania.

F1 going up did not automatically mean better gameplay.

## The Final State

This is no longer “LLMs try osu.”

It is audio-to-timed-event prediction under strict timing windows.

The current state is:

- a 50+ GB dataset
- a working audio pipeline
- leakage removed from the feature pipeline
- decent press prediction
- weak but real release prediction
- hold modelling that kind of works but is unstable
- replay generation present but not fully integrated into training
- checkpointed training that can resume instead of restarting from scratch

It went from a joke to an actual sequence-modeling problem.

## Failure Modes Worth Caring About

- label leakage re-entering the feature pipeline
- frame-level metrics looking good without corresponding gameplay value
- release prediction staying weak because audio cues do not sharply mark release timing
- long training times making iteration too slow without checkpoints or reduced validation loops
