# slyml

This tutorial is of general interest for `slurm` users, but slyml greatly simplified frequent conversion of image stacks to surface meshes needed for the [3DXP project](https://github.com/Rhoana/3dxp) in [Scalable Interactive Visualization for Connectomics](http://www.mdpi.com/2227-9709/4/3/29/pdf). Slyml is developed and maintained [in the 3DXP repository](https://github.com/Rhoana/3dxp/blob/master/TASKS/readme.md). Reminder: `Slyml` needs python2 (>=2.6) and slurm (>=14.11).

> A B C it's easy,
> It's like counting up to 3.
> Sing a simple melody:
> That's how easy slurm can be!

— The Jackson 5, ABC _(edited for clarity)_

## Lesson 0: Needs

Let's say our script `something.sbatch` tries to find some words spoken in video streams. Getting timestamps for the first lyric `"Itʼs easy as A, B, C."` will help find timestamps for the second lyric `"As simple as do, re, mi"`, and so on and so on as the song continues. If only the lyric changes each time we call our script, this may be `something.sbatch`:

```bash
#!/bin/bash
# Find videos with given lyric
python find_videos.py --lyric="$lyric"
```

Our `something.sbatch` just calls `find_videos.py` with a single `lyric`, but we can write a file called `song.yaml` that:

1. Lets us quickly update the lyrics by:
    * changing words without rearranging lyrics
    * rearranging lyrics without changing words
2. Tells `slurm` to handle lyrics in a certain order 

```yaml
# Here we use nodes in a slurm partition called "general" 
# to run "something.sbatch" on a sequence of 3 lyrics
#
Main:
    lyric: "As simple as {A} {B} {C}"
    Needs:
        lyric: "Itʼs easy as A, B, C."
    For:
        lyric: "A B C, {A} {B} {C}: Thatʼs {THAT}"
        Inputs:
            THAT: "how easy {IT} can be."
            A: 1
            B: 2
            C: 3
Default:
    Slurm: ./do/something.sbatch
    Flags: [partition, time]
    partition: general
    Exports: [lyric]
    time: "1:00"
    Inputs:
        IT: love
        A: do,
        B: re,
        C: mi.
```
Running `python song.yaml` queues each lyric in the correct order. Our `something.sbatch` then handles each `lyric` in order one after another. **Here is the standard output**:
```yaml
A/song\
2A/song\
Exports:
  lyric: Itʼs easy as A, B, C.
Slurm: something.sbatch
2A/song/
.......
Exports:
  lyric: As simple as do, re, mi.
Slurm: something.sbatch
A/B/song\
Exports:
  lyric: 'A B C, 1 2 3: Thatʼs how easy love can be.'
Slurm: something.sbatch
AB/song Needs:
- 1x A/song
A/B/song/
........
A/song Needs:
- 1x 2A/song
A/song/
......
*~*~*~*~*~*~*~*~*~*~*~*~*~*~*~*
 SUCCESS: Scheduled all jobs.
______________________________
```
You'll notice `slyml.py` gives each line in `song` a name: 

```
A/song
  needs 2A/song
  for AB/song
```
`AB`—the last needs `A`—the focus, which needs `2A`—the first. I'll annotate to the original:

```yaml
Main: &A-focus
    lyric: "As simple as {A} {B} {C}"
    Needs: &2A-first
        lyric: "Itʼs easy as A, B, C."
    For: &AB-last
        lyric: "A B C, {A} {B} {C}: Thatʼs {THAT}"
```

If you `watch squeue`, you'll see that `2A` runs first:
```
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
    00000001_[0%1]   general    A/son username PD       0:00      1 (Dependency)
    00000002_[0%1]   general   AB/son username PD       0:00      1 (Dependency)
        00000000_0   general   2A/son username  R       0:01      1 node_id12345
```
When `2A` finishes, `A` will begin:
```
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
    00000002_[0%1]   general   AB/son username PD       0:00      1 (Dependency)
        00000001_0   general    A/son username  R       0:00      1 node_id12345
```
In the same way, `AB` will begin when `A` finishes.

## Lesson 1: Anchors

Let's write a `code.yaml` that copies the anchor `&advice` with the alias `*advice`.

With this code, we use the format `"A B C, {A} {B} {C}: Thatʼs {THAT}"` in two ways:
1. `python slyml.py code.yaml` looks up `A B C, 1 2 3: Thatʼs how easy love can be.` as the last step like `song.yaml`.
2. `python slyml.py code.yaml code` only looks up `A B C, always be coding: Thatʼs bad advice` as the only step.

```yaml
# slyml.py v2.2
#
Main:
    lyric: "As simple as {A} {B} {C}"
    Needs:
        lyric: "Itʼs easy as A, B, C."
    For: &advice
        lyric: "A B C, {A} {B} {C}: Thatʼs {THAT}"
        Inputs:
            THAT: "how easy {IT} can be."
            A: 1
            B: 2
            C: 3
code: 
    <<: *advice
    Inputs:
        THAT: "bad advice!"
        A: always
        B: be
        C: coding
Default:
    Slurm: ./do/something.sbatch
    Flags: [partition, time]
    partition: general
    Exports: [lyric]
    time: "1:00"
    Inputs:
        IT: love
        A: do,
        B: re,
        C: mi.
```
