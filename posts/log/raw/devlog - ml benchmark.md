## DEVLOG – Turning That Mania Benchmark Mess Into Actual ML

Hey, so yeah, this started out as a total API nightmare. I had this pipeline going: grab the .osz file, pull out the notes, turn them into frames, batch them, chuck it at some LLM API, get back whatever nonsense it spits out, convert that to a replay, score it, and then just kinda stare at the screen feeling dumb.

None of those models got above like 2% accuracy. Makes sense though, they're not built for timing or predicting sequences that actually matter. They just barf out stuff that looks structured but misses the rhythm completely. It's like asking a poet to do calculus.

That's when it hit me: screw this "LLMs play osu" vibe. Let's make it a legit ML benchmark instead.

Pivoted hard. Now it's about building my own model, training it from scratch, measuring the crap out of it, and tweaking until it doesn't suck. No more API begging. Just straight-up machine learning.

## Dataset Scaling Nightmare

I Needed way more data to make this work. I remembered when i played Etterna it had tons of free packs, sounded perfect. But their download speed caps at about 1MB/s, which meant I'd be waiting around for days and no one seeded the torrents either, soooo

Pivoted again. Threw together an osu API v2 client to snag beatmap metadata. API doesn't let you download directly, which is annoying, so I hacked around with cookie auth for direct grabs. unreliable, but it did the job for the 2 maybe 3 times i used it

Then I remembered osu has song packs like etterna with a higher download speed at around ~50MB/s. Instead of scraping one by one like a noob, I hit the packs endpoint and automated the whole thing.

I wrote a script to:
- List out all the packs
- Fetch beatmapsets and show the links through osuapi v2
- Download the .zip files
- Dump everything into a data folder
- Dedupe by beatmapset ID
- Keep track of unique songs

Ended up with 336 packs, around 6.6k beatmapsets, 5.8k unique songs, and about 51GB of raw data. Went from zero to overload real quick.

## My First ML Try

I got the dataset built, split it by maps instead of frames to avoid any leakage; because that's just embarrassing.

Trained a tiny TCN model. Press F1 score? Straight 1.0; I said “no way I’m that good”

Spoiler: I wasn't. Turns out the features for lanes 0-3 were literally the same as the press labels. Same frame and same spots. Model just copied input to output. So I basically invented a fancy photocopier.

Releases were worse because the features marked holds as 1 the whole time, but labels only ping 1 on the exact release frame. So it cheated on presses, x_x on releases. Total leakage disaster.

## Audio rewrite

Decided: The model must predict from sound, not the chart.

Redid the feature pipeline:
- Pull audio from .osz using the per-difficulty AudioFilename
- Load with librosa at 22050Hz
- 10ms hop length
- 80-bin mel spectrogram
- Onset strength and RMS
- Aggregate to 20ms grid to match labels
- Save as float16
- Normalize based on train split stats only

Features end up at 82 dims pooled, or 164 stacked. Zero note stuff allowed. Burned it all.

## Early Audio Results (On 200 Maps)

Training felt way more legit now.

Press F1 around 0.35, releases 0.06 to 0.09, holds about 0.25.

With a ±1 frame wiggle room: presses up to 0.42, releases 0.09, holds 0.17.

Not blowing anyone's mind, but it's honest work. Model's picking up onsets okay, but releases? Audio doesn't scream "stop now" super clearly. And it overdoes holds like it's trying too hard.

## Eval Wake-Up Call

Micro-F1 hovers 0.25 to 0.33 based on thresholds. Precision's meh, recall's better. Releases drag it down, holds are all over.

Copy-last baseline looks stupid good with tolerance, but frame eval is kinda pointless anyway.

Hit me: classifying frames isn't the same as playing mania. Replays care about hits, not per-frame matches.

So F1 going up doesn't auto-mean better gameplay.

## Infra Struggles

Training the full set? Hours on end, roughly ~10hrs

Fixed it with checkpoints for model and optimizer, resume whenever. Tiny validation for quick checks. Maybe cloud GPU later if I get desperate.

## What This Turned Into

Ditched the "LLMs try osu" gimmick. Not even "AI plays game" anymore.

It's audio to timed event prediction, with super strict windows. And it actually learns stuff.

## Where It's At Now

I'm sitting on a 50GB dataset, audio pipeline's working, no leakage left, presses are decent, releases exist but weak, holds kinda work. Replay gen's there but not looped into training yet. And training picks up where it left off.

Went from joke to actual sequence problem. Still rough around the edges but hey, it's not bad for a dropout.