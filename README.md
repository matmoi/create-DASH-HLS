# create-DASH-HLS
A tutorial to generate fMp4 files compatible with dash and HLS

To illustrate with an example, we use the film [Elephants Dream](https://orange.blender.org/), which is under Creative Commons license, and more specifically this [mp4 version](http://ia600209.us.archive.org/20/items/ElephantsDream/ed_hd.mp4).

## Requirements

- [ffmpeg](https://www.ffmpeg.org/download.html) is needed to prepare media files for ABR streaming, including `ffprobe` command line tool to display information about media streams.
- [bento4](https://www.bento4.com/downloads/) is also required to structure media bitstreams as expected. For now, I recommend using this [fork](https://github.com/matmoi/Bento4) which contains a bunch of fixes to handle webvtt subtitles properly. It's not required to build it from scratch, one can simply replace `Bento4\utils\mp4-dash.py` with this [version](https://raw.githubusercontent.com/matmoi/Bento4/master/Source/Python/utils/mp4-dash.py).

## Video streams
The video track of `ed_hd.mp4` will be used as the mezzanine media file. A mezzanine file is a lightly compressed master file that will stand up to making additional compressed versions. Let's have a look at how the bitstream is structured, using `ffprobe` :

```
ffprobe -show_frames -select_streams v:0 -print_format csv=print_section=0:item_sep=; -show_entries frame=key_frame,pkt_pts_time,coded_picture_number ed_hd.mp4 > ed_hd.csv
```

First, terminal output shows some useful information, more precisely :

```
 Duration: 00:10:53.85, start: 0.000000, bitrate: 829 kb/s
    Stream #0:0(und): Video: h264 (Constrained Baseline) (avc1 / 0x31637661), yuv420p, 640x360, 696 kb/s, 24 fps, 24 tbr, 12288 tbn, 48 tbc (default)
```

Movie lasts `10:53.85` in total, hence `653.85s` , video stream is encoded using h264 constrained baseline with an avg bitrate of `696kbps`, a resolution of `640x360` and a framerate of `24fps`. Now let's have a look at `ed_hd.csv`, which contains per frame information. First column indicates if the frame is a key frame, second is the packet presentation timestamp, aka. PTS, and last shows the coded picture counter. The file contains `15691` lines, one per frame, as we can expect `(15691+1)/24 = 653.83` (number of frames + 1 / framerate = stream duration). There are `147` key frames, meaning one every `4.45s` roughly. Maximum time range between two consecutives key frame is `10.416667s`, we can conclude that max gop size was set to 250 (`10.416667 * 24fps`), which is way too much for our purpose. So, we must re-encode our mezzianine video file to serve us better, with a fix gop size of `2s`. This is what's broadly used in the industry, as a good trade-off between coding efficiency and the ability to quickly react depending on network condition changes. We use a two pass encoding as following:

```
ffmpeg -y -i ed_hd.mp4 -c:v libx264 -x264-params keyint=48:min-keyint=48:scenecut=-1:nal-hrd=cbr -b:v 600k -bufsize 1200k -maxrate 600k -profile:v high -level 4.2 -pass 1 -an -f mp4 NUL && ffmpeg -i ed_hd.mp4 -c:v libx264 -x264-params keyint=48:min-keyint=48:scenecut=-1:nal-hrd=cbr -b:v 600k -bufsize 1200k -maxrate 600k -profile:v high -level 4.2 -pass 2 -an ed_hd_640x360.mp4
```

We specify a gop size of `48` frames (every `2s`) with `keyint=48:min-keyint=48`, no scene cut detection (to avoid unexpected key frames) and constant bitrate mode targetting `600kbps`. As you can see, the target bitrate is slightly lower than for the mezzianine file, this is because we use h264 high profile which provides a much better compression rate than constrained baseline, even if we drastically reduce the gop size it should not impact video quality. We also specify that our HRD model has a max size of twice the bitrate, meaning that each fragment of `2s` cannot exceed the target bitrate.

For this experiment, we'll also generate two additionnal low-resolution video streams of `480x270` and `320x180` respectively, with corresponding bitrates of `400kbps` and `200kbps` (chose arbitrarily for this example):

```
ffmpeg -y -i ed_hd.mp4 -vf scale=w=480:h=270 -c:v libx264 -x264-params keyint=48:min-keyint=48:scenecut=-1:nal-hrd=cbr -b:v 400k -bufsize 800k -maxrate 400k -profile:v high -level 4.2 -pass 1 -an -f mp4 NUL && ffmpeg -i ed_hd.mp4 -vf scale=w=480:h=270 -c:v libx264 -x264-params keyint=48:min-keyint=48:scenecut=-1:nal-hrd=cbr -b:v 400k -bufsize 800k -maxrate 400k -profile:v high -level 4.2 -pass 2 -an -movflags frag_keyframe ed_hd_480x270.mp4
```

```
ffmpeg -y -i ed_hd.mp4 -vf scale=w=320:h=180 -c:v libx264 -x264-params keyint=48:min-keyint=48:scenecut=-1:nal-hrd=cbr -b:v 200k -bufsize 400k -maxrate 200k -profile:v high -level 4.2 -pass 1 -an -f mp4 NUL && ffmpeg -i ed_hd.mp4 -vf scale=w=320:h=180 -c:v libx264 -x264-params keyint=48:min-keyint=48:scenecut=-1:nal-hrd=cbr -b:v 200k -bufsize 400k -maxrate 200k -profile:v high -level 4.2 -pass 2 -an -movflags frag_keyframe ed_hd_320x180.mp4
```

Note that the`-an` option drops the audio track, which will be handled in next section.

Now that we have transcoded our video streams with the expected format, it's possible to generate the fragmented version of our mp4 files with the commands :

```
mp4fragment ed_hd_320x180.mp4 ed_hd_320x180_fragments.mp4
mp4fragment ed_hd_480x270.mp4 ed_hd_480x270_fragments.mp4
mp4fragment ed_hd_640x360.mp4 ed_hd_640x360_fragments.mp4
```

## Audio streams

Source file `ed_hd.mp4` contains the original audio track aleady. Let's have a look at it :

```
Stream #0:1(und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, stereo, fltp, 128 kb/s (default)
```

We will simply extract the audio track from this file by using :

```
ffmpeg -i ed_hd.mp4 -vn -c:a copy -metadata:s:a:0 language=eng ed_hd_english.mp4
```

For this audio track, language is set to english with `-metadata:s:a:0 language=eng`. In order to simulate audio multitracking, we'll add an additionnal french audio track which is a degraded version of the original. Not really usefull but it has the merit of showing how to deal with multiple audiotracks.

```
ffmpeg -i ed_hd.mp4 -vn -b:a 24k -c:a aac -metadata:s:a:0 language=fre ed_hd_french.mp4
```

If you listen to `ed_hd_french.mp4`, you'll ear indeed how crappy it sounds. This low quality audio file, as well as the original version, can now be fragmented as required by ABR streaming using :

```
mp4fragment --fragment-duration 2000 ed_hd_french.mp4 ed_hd_french_fragments.mp4
mp4fragment --fragment-duration 2000 ed_hd_english.mp4 ed_hd_english_fragments.mp4
```

As for video tracks, we want fragments of `2s`, specified with `--fragment-duration 2000`.

## Subtitles

We simply consider two [webvtt](https://w3c.github.io/webvtt/) files to illustrate this example :
- https://github.com/matmoi/create-DASH-HLS/raw/master/examples/elephants-dream-subtitles-de.vtt
- https://github.com/matmoi/create-DASH-HLS/raw/master/examples/elephants-dream-subtitles-en.vtt

> As I'm typing, Bento4 has a [bug](https://github.com/axiomatic-systems/Bento4/issues/150) which prevents using external webvtt files with on-demande profile. I proposed a fix in https://github.com/matmoi/Bento4.

## Generate DASH/HLS files for streaming

```
mp4dash --verbose --profiles=on-demand --hls --subtitles -o ElephantsDream ed_hd_640x360_fragments.mp4 ed_hd_480x270_fragments.mp4 ed_hd_320x180_fragments.mp4 ed_hd_english_fragments.mp4 ed_hd_french_fragments.mp4 [+format=webvtt,+language=eng]elephants-dream-subtitles-en.vtt [+format=webvtt,+language=deu]elephants-dream-subtitles-de.vtt
```

## Other samples

- [Bitmovin](https://bitmovin.com/mpeg-dash-hls-examples-sample-streams/) provides a bunch of samples for DASH and HLS.
https://bitmovin.com/mpeg-dash-hls-examples-sample-streams/
- [Akamai](http://players.akamai.com) also proposes several DASH and HLS streams.