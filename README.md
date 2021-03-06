# Sly Markup Language

| Where? | What? |
| ---  | --- |
| [• Inputs](#lesson-0-inputs) [• Needs](#lesson-1-needs) [• Alias](#lesson-2-alias) [• Array](#lesson-3-array)  | This tutorial is of general interest for `slurm`[†](#slurm-sbatch) users, but `slyml.py` is so far only used to convert image stacks to meshes needed for the [3DXP project](https://github.com/Rhoana/3dxp) in [Scalable Interactive Visualization for Connectomics](http://www.mdpi.com/2227-9709/4/3/29/pdf). Slyml is developed and maintained [in the 3DXP repository](https://github.com/Rhoana/3dxp/blob/master/TASKS/readme.md). Reminder: `slyml.py` needs `python2 (>=2.6)` and `slurm (>=14.11)`. It's easy to [get started](#installation)!|

> A B C it's easy,
> It's like counting up to 3.
> Sing a simple melody:
> That's how easy slurm can be!

— The Jackson 5, ABC _(edited for clarity)_

Assume `something.sbatch` is in the same directory as this `abc.yaml` file:

```yaml
# Here we use nodes in a slurm partition called "general" 
# to run "something.sbatch" on a sequence of 2 lyrics
#
Main:
    sbatch_argument: "As simple as do, re, mi."
    For:
        sbatch_argument: "Itʼs easy as A, B, C."
    Exports: [sbatch_argument]
    Slurm: ./something.sbatch
    Flags: [partition, time]
    partition: general
    time: "1:00"
```

Then `python slyml.py abc.yaml` will schedule `something.sbatch` to run twice in sequence:
   - First, the `sbatch_argument` is set to `"As simple as do, re, mi."` as a variable in `something.sbatch`
   - After that's done, `something.sbatch` will run again with `sbatch_argument` as `"Itʼs easy as A, B, C."`

If we want to handle both lyrics simultaneously, we just write:

```yaml
Main:
    For:
      - sbatch_argument: "As simple as do, re, mi."
      - sbatch_argument: "Itʼs easy as A, B, C."
```

instead of:

```yaml
Main:
    sbatch_argument: "As simple as do, re, mi."
    For:
        sbatch_argument: "Itʼs easy as A, B, C."
```

## Lesson 0: Inputs

Let's say our script `something.sbatch` tries to find some words spoken in video files with `find_videos.py`. If only the `lyric` changes each time we call our script, `something.sbatch` may look like this:

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
    Constants:
        IT: love
        A: do,
        B: re,
        C: mi.
```
Notice how the last task in `Main` has `Inputs` like this:
```
  Inputs:
      THAT: "how easy {IT} can be."
      A: 1
      B: 2
      C: 3
```
that overwrite or inherrit from the `Constants` in `Default`:
```
  Constants:
      IT: love
      A: do,
      B: re,
      C: mi.
```
Key takeaways:
* The `Inputs` can format the `lyric` of any task
* The `Constants` can override or format the `Inputs`
* Each task sends `Exports: [lyric]` to our `something.sbatch`
* Each task sends `Flags: [partition, time]` to slurm's `sbatch`

## Lesson 1: Needs

Running `python slyml.py song.yaml` queues each lyric in the correct order. Our `something.sbatch` then handles each `lyric` in order one after another. **Here is the standard output** of `slyml.py`:
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
`AB`—the last needs `A`—the focus, which needs `2A`—the first. We can note these names in the yaml file:

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
        00000001_0   general    A/son username  R       0:01      1 node_id12345
```
In the same way, `AB` will begin when `A` finishes.

## Lesson 2: Alias

Let's write a `code.yaml` that copies the anchor `&advice` with the alias `*advice`.

With this code, we have two options based on `"A B C, {A} {B} {C}: Thatʼs {THAT}"`:
1. `python slyml.py code.yaml` looks up `A B C, 1 2 3: Thatʼs how easy love can be.` as the last of three steps.
2. `python slyml.py code.yaml code` only looks up `A B C, always be coding: Thatʼs bad advice` as the only step.

```yaml
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
    Constants:
        IT: love
        A: do,
        B: re,
        C: mi.
```
This is a huge benefit for code reuse: all your jobs can differ slightly, in many ways, even fundamental structure! But with `slyml.py` you never need to copy any of the unchanged bits of logic, format, or input.

## Lesson 3: Array

| | | Imagine you have a lot of tasks in a list. |
| --- |--- | --- |
| If you're Santa Claus, you need a high throughput solution that processes billions of gifts simultaneously. | | [![oh no](http://img.hoff.in/slyml/wcn-xmas.png?q=123)](http://webcomicname.com/post/154820035714) |

If we have 12 CPUs free on our cluster nodes, we can handle 12 unique groups of gifts simultaneously. We can write a new `something.sbatch` that assumes `$SLURM_ARRAY_TASK_ID` gives the position of the requested gift in a group of gifts.

```bash
#!/bin/bash
GIFTS=("Partridge in a pear tree" "Turtle doves" "French hens" "Calling birds" "Golden rings" "Geese a-laying" "Swans a-swimming" "Maids a-milking" "Ladies dancing" "Lords a-leaping" "Pipers piping" "Drummers drumming")
DAY=$SLURM_ARRAY_TASK_ID
GIFT=${GIFTS[$DAY]}

# Find videos with a given christmas lyric
python send_gifts.py --gift="$DAY $GIFT"
```

By setting `Runs: 12` in `gifts.yaml`, we can run  `python slyml.py gifts.yaml` to send gifts for all 12 days of Christmas simultaneously over 12 CPUS on our Slurm cluster.

```yaml
Main:
    Slurm: ./do/something.sbatch
    Flags: [partition, time]
    partition: general
    time: "1:00"
    Runs: 12
```
If needed, `$SLURM_ARRAY_TASK_COUNT` will be `12` across all jobs.
Technically, this `gifts.yaml` runs the following 12 commands in parallel:
```bash
python send_gifts.py --gift="0 Partridge in a pear tree"
python send_gifts.py --gift="1 Turtle doves"
python send_gifts.py --gift="2 French hens"
...
python send_gifts.py --gift="11 Drummers drumming"
```

I leave fixing the off-by zero error as an excercise in bash.
Condolences to the partridge.

## Footnotes
#### Slurm Sbatch
Installing [Slurm](https://slurm.schedmd.com/quickstart.html) lets you run processes in parallel over a [cluster](https://en.wikipedia.org/wiki/Computer_cluster) of computers running [linux](https://slurm.schedmd.com/platforms.html). Slurm includes `sbatch`, which executes scripts like `something.sbatch` as "slurm jobs" with [optional flags](https://slurm.schedmd.com/sbatch.html) such as `--array` for parallel jobs or `--dependency` for serial dependencies between jobs.

## Installation

1. Get access to a slurm cluster or try to [set one up](https://slurm.schedmd.com/quickstart_admin.html) with the extra supercomputers around the house.
2. Open a login shell on a slurm cluster node with access to `sbatch`(>=14.11) and `python`(>2.6)
3. `git clone https://github.com/Rhoana/3dxp.git`
4. run `python 3dxp/TASKS/slyml.py` on any slyml.yaml file. 

[![I figured starting a slurm cluster would be easy. After all, I've got tons of computers!](http://img.hoff.in/slyml/xkcd-slurm.png?q=1234)](https://xkcd.com/1506/)
