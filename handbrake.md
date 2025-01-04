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
  /Volumes/POIROT_SERIES*_DISC* \
  /Volumes/movies/poirot/
```

We copy the files so that Handbrake isn't IO-limited: the DVD player has a
meager throughput of 4 MiB/s.

### Prepare the Subtitles

- Browse to <https://www.opensubtitles.org/>
- Log in
- Search for "Poirot"
- Click the first episode of Season 1
- Sort by subtitle rating ("Kynitos")
- Download the first English subtitle as "Poirot 1 1.srt"
- Convert to NTSC:

```bash
less /tmp/*.srt # make sure you downloaded the English version
chardetect /tmp/*.srt # check the encoding
SEASON=1
for EPISODE in $(seq 1 10); do
   TITLE="Poirot ${SEASON} ${EPISODE}"
  ~/bin/pal_speedup_srt.rb < /tmp/"$TITLE".srt > /Volumes/movies/poirot/srt/"$TITLE".srt
done
```

Example: to convert ISO-8859-1 encoding to UTF-8

```bash
for SRT in /tmp/Poirot\ 2\ *.srt; do
  iconv -f ISO-8859-1 -t UTF-8 "$SRT" > "$SRT-UTF-8"
  mv  "$SRT"{-UTF-8,}
done
```

### Configure Handbrake:

- Source: Browse to `/Volumes/POIROT_SERIES1_DISC1`
- Title: select the first fifty-minute title
- Save as: `/Volumes/movies/Poirot 1 1.m4v`
- Subtitles
  - Tracks → Add External Subtitles Track...
  - Select /Volumes/movies/poirot/srt/Poirot 1 1.srt
  - Check "default"
- Click "Add to Queue"
- Click "Start Queue"

#### From the CLI

For old-school SubRip (`.srt`) subtitle files:

```bash
DISC=1
SEASON=1
EPISODE_OFFSET=0
for TITLE in 1 2 3 4; do
  EPISODE=$(( TITLE + EPISODE_OFFSET ))
  handbrakecli \
    --preset="Fast 1080p30" \
    --subtitle="scan" \
    --subtitle-forced \
    --subtitle-burned \
    --srt-file="/Volumes/movies/poirot/srt/Poirot $SEASON $EPISODE.srt" \
    --srt-lang="eng" \
    --srt-codeset="UTF-8" \
    --srt-default \
    --title $TITLE \
    -i /Volumes/movies/poirot/POIROT_SERIES${SEASON}_DISC${DISC}/ \
    -o /Volumes/movies/Poirot\ $SEASON\ $EPISODE.m4v
done
```

For newer Advanced SubStation Alpha (`.ass`) subtitle files:


```bash
DISC=2
SEASON=2
EPISODE_OFFSET=3
for TITLE in 1 2 3; do
  EPISODE=$(( TITLE + EPISODE_OFFSET ))
  handbrakecli \
    --preset="Fast 1080p30" \
    --subtitle="scan" \
    --subtitle-forced \
    --subtitle-burned \
    --ssa-file="/Volumes/movies/poirot/srt/Poirot $SEASON $EPISODE.ass" \
    --ssa-lang="eng" \
    --ssa-default \
    --title $TITLE \
    -i /Volumes/movies/poirot/POIROT_SERIES${SEASON}_DISC${DISC}/ \
    -o /Volumes/movies/Poirot\ $SEASON\ $EPISODE.m4v
done
```
### References

- <https://en.wikipedia.org/wiki/576i#PAL_speed-up>
- <https://en.wikipedia.org/wiki/SubRip>
- <https://en.wikipedia.org/wiki/Windows-1252>
