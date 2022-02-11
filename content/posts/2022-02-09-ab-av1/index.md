---
title: "AV1 encoding: ab-av1"
---
In [my previous post](/posts/av1-p1) I looked at video encoding using svt-av1 with **crf** & **preset** settings and using VMAF to rate the resultant quality. Av1 encoding is reasonably fast now, but encoding a full video then checking the VMAF potentially multiple times to find a good crf & preset... that is going to be _not so fast_.

## Sample encoding
My answer was to cut a small section of the _vid.mp4_ original video, say 20s long, and encode that. It's obviously much faster encoding a short clip, it's also similarly much faster to measure the VMAF. From the clip we can calculate/guess the size of the fully encoded output too. However, the short clip may not be representative of the full video and produce inaccurate VMAF/size.

But we can improve the accuracy by taking multiple samples considering all of them. It's still way faster to encode & VMAF a bunch of 20s samples than to do the full video.

```text
vid.mp4
+--------------------------------------------------------------------------------+
|               |A|                   |B|                   |C|                  |
+--------------------------------------------------------------------------------+
                 |                     |                     |
                 v                     v                     v
 cut 20s     vid.a.mp4             vid.b.mp4             vid.c.mp4
                 |                     |                     |
                 v                     v                     v
 encode     vid.a.av1.mp4         vid.b.av1.mp4         vid.c.av1.mp4     
```
* Cut _vid.mp4_ into 3 20s samples spaced throughout the full video (a, b & c). This is done with ffmpeg and is a comparatively instant operation.
* Encode each sample using svt-av1.
* Calculate VMAF score for each sample.

Then from the samples we can calculate the mean VMAF and size diff. We can use this to predict what the full video size & VMAF would be after a (potentially lengthy) full encode.

While it's easy enough to cut a sample using ffmpeg the whole process got complicated enough that I wrote a CLI program to help me.

## ab-av1
**ab-av1** ([github](https://github.com/alexheretic/ab-av1#readme), [AUR](https://aur.archlinux.org/packages/ab-av1)) is a rust CLI binary that I wrote to handle sample encoding _(and more)_. It calls out to ffmpeg & svt-av1 as child processes to do the actual cutting, encoding and VMAFing.

ab-av1 has functionality split into subcommands. The first one I wrote handles sample encoding.

### ab-av1 sample-encode
> Encode short video samples of an input using provided crf & preset. This is much quicker than full encode/vmaf run.
```text
ab-av1 sample-encode [OPTIONS] -i <INPUT> --crf <CRF> --preset <PRESET>
```

Let's give it a bash on the same _vid.mp4_ we used previously.

<video src="ab-av1-sample-encode.mp4" poster="ab-av1-sample-encode.avif" playsinline controls></video>

So from the 3 samples we can predict VMAF **96.36** and a **59% encoded size**. Since already fully encoded this video with these settings we know that real result was VMAF **96.04** with **66%** size. So we can see the predictions are indeed approximate. But the big difference is we did our prediction in **19 seconds** vs ~13 minutes.

We can always up the sample count if we suspect our video to be more variable. E.g. if I'd used 8 samples above it would have taken 47 seconds to predict VMAF **96.04** with **68%** size, which is _very_ close to the truth and still a lot faster than encoding the whole thing.

With **sample-encode** we now have a fast & convenient way to how analyse any svt-av1 crf and preset setting. We can now start using it to search for the highest crf that achieves a given VMAF score. Naturally ab-av1 has a subcommand for this.

### ab-av1 crf-search
> Interpolated binary search using sample-encode to find the best crf value delivering **min-vmaf** & **max-encoded-percent**.
```text
ab-av1 crf-search [OPTIONS] -i <INPUT> --preset <PRESET>
```

Let's try it out by searching for the best crf for _vid.mp4_ to get VMAF 94.

![](ab-av1-crf-search.png)

The command called sample-encode 4 times as it searched for the highest crf to satisfy our VMAF constraint. It tells us that if we can up our crf to **36** to get our desired quality. Higher crf means our output is will be even smaller. It also predicts how long the full encode will take, ~6 minutes, which we already know is pretty close to the truth.

Because sample-encode itself is so fast the entire crf-search took only around a minute. Of course this will slow down with lower presets, but should always be proportionally much quicker than doing a full encode.

## Evaluating svt-av1 presets
ab-av1 provides a way to take an objective look at svt-av1 presets and to do it in _a single human lifetime_. We already know lower presets are higher quality and take longer, but how high and how long?

Well lower presets are _very_ slow indeed. It's also true that all presets produce roughly the same size output on a given crf. So we need to get a VMAF score to show the true value.

**crf-search** can do this, if the preset gives better quality we should be able to find a higher crf value to satisfy our VMAF constraint.

![](ab-av1-presets.png "For presets under 4 even the sample encodes take forever")

And indeed this is the case. We can use higher crf values to achieve VMAF 94 as we use lower presets. From this example preset **6** seems a good compromise between speed and quality.

## ab-av1: extras
**sample-encode** & **crf-search** are the "guts" of ab-av1, but it also comes with some other commands that make use of, or can be used alongside, those. They are at least more ergonomic that using ffmpeg directly and have nice progress bars.

* **auto-encode** _Automatically determine the best crf to deliver the min-vmaf and use it to encode a video._
  ```text
  ab-av1 auto-encode [OPTIONS] -i <INPUT> --preset <PRESET>
  ```
* **encode** _Simple invocation of ffmpeg & SvtAv1EncApp to encode a video._
  ```text
  ab-av1 encode [OPTIONS] -i <INPUT> --crf <CRF> --preset <PRESET>
  ```
* **vmaf** _Simple full calculation of VMAF score distorted file vs original file._
  ```text
  ab-av1 vmaf --original <ORIGINAL> --distorted <DISTORTED>
  ```

If you have ideas to improve ab-av1, come on over and raise an issue at [alexheretic/ab-av1](https://github.com/alexheretic/ab-av1).

## Further reading: av1an
You should also be aware of the more ambitious project [av1an](https://github.com/master-of-zen/Av1an), it's also a rust CLI wrapper that controls child processes. It also supports encoding to a "target-quality" VMAF score. 

However, I found it's VMAF analysis to be too slow for my patience, hence my investigation into sample encoding & VMAF. I also find svt-av1 to do good enough threading on its own, so don't benefit too much from the chunk-based concurrent encoding av1an provides _(though libaom & other encoders are perhaps a different story)_.
