## Handbrake

In this example, we use Handbrake to convert our Poirot DVD collection so we
can watch it on our iPhones.

### Prepare the Source

First, mount our `/Volumes/movies`:

- Finder → ⌘ k → afp://nas.nono.io → movies
- Insert the DVD
- Press ⏹️  on the DVD player so the trailers don't contend with the copying
- Copy the files to the network volume

```bash
rsync \
  -avH \
  -W \
  --ignore-errors \
  --partial \
  --stats \
  --progress \
  /Volumes/POIROT_SERIES1_DISC1 \
  /Volumes/movies/poirot/
```

We copy the files so that Handbrake isn't IO-limited: the DVD player has a
meager throughput of 4 MiB/s.

### Prepare the Subtitles

- Browse to <https://www.opensubtitles.org/>
- Log in
- Search for "Poirot"
- Click the first episode of Season 1
- Sort by subtitle rating
- Download the first English subtitle as "Poirot 1 1.srt"
- Convert to NTSC:

```bash
EPISODE="Poirot 1 1"
~/bin/pal_speedup_srt.rb < ~/Downloads/"$EPISODE".srt > /Volumes/movies/poirot/srt/"$EPISODE".srt
```

### Configure Handbrake:

- Source: Browse to `/Volumes/POIROT_SERIES1_DISC1`
- Title: select the first fifty-minute title
- Save as: `/Volumes/movies/Poirot 1 1.m4v`
- Subtitles
  - Tracks → Add External Subtitles Track...
  - Select /Volumes/movies/poirot/srt/Poirot 1 1.srt
- Click "Add to Queue"
- Click "Start Queue"

#### From the CLI

```bash
handbrakecli \
  --preset="Fast 1080p30" \
  --subtitle="scan" \
  --subtitle-forced \
  --subtitle-burned \
  --srt-file="/Volumes/movies/poirot/srt/Poirot 1 1.srt" \
  --srt-lang="eng" \
  --srt-codeset="UTF-8" \
  --srt-default \
  -i /Volumes/movies/poirot/POIROT_SERIES1_DISC1/VIDEO_TS/VTS_01_1.VOB \
  -o /Volumes/movies/Poirot\ 1\ 1.m4v
```

### References

- <https://en.wikipedia.org/wiki/576i#PAL_speed-up>
- <https://en.wikipedia.org/wiki/SubRip>
- <https://en.wikipedia.org/wiki/Windows-1252>
