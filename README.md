# VHS Capture Notes
Notes on digitizing VHS tapes

## Capturing
### Hardware
#### Capture card
Finding hardware for capturing from legacy inputs is a small challenge.
[Modern capture cards](https://www.amazon.com/Elgato-Game-Capture-4K60-MK-2/dp/B07VWXCXM) don't have
legacy inputs. There are [well-reviewerd](https://www.amazon.com/Elgato-Video-Capture-Digitize-iPad/dp/B0029U2YSA)
USB 2.0 cards, but Linux support tends to be worse, even Windows support in tools like Virtualdub2 is worse,
USB 2.0 can be more finicky than PCIe, and moving raw 30fps 720x480 video takes around 256Mbps. USB 2.0 runs at
480 Mbps. Using over half the throughput on a high-latency (at least comparted to PCI) interface for a realtime
application is asking for dropped frames. USB capture cards tend to encode as MPEG2 (the one above might use x264)
before sending the data over the wire. Even if that's your target format, the quality won't be as good as with `ffmpeg`
and targeting a file size or quality. You'll probably also want to do a little post-processing like cropping to
remove head-switch noise, so again, raw is the way to go.

I settled on a  Hauppauge ImpactVCB-e, but I wasn't thrilled with it. In the early 2000s, Hauppauge was known for
making good analog capture cards with good Linux support. Linux support was ok, but the Windows drivers were old,
and I wasn't able to use the card with VMWare PCI Passthrough or a Thunderbolt 3 enclosure.

#### VCR
[lordsmurf](http://www.digitalfaq.com/forum/news/3230-video-advice-lordsmurf.html) has an [amazing guide on VCR selection](
http://www.digitalfaq.com/forum/video-restore/1567-vcr-buying-guide.htm).

### Software guide
#### Linux
linuxtv.org has a [detailed wiki explaining VHS capture](https://linuxtv.org/wiki/index.php/V4L_capturing) on
Linux. It's a long, but worthwhile read. The only bits that were missing were how to choose which
input source on the device (svideo, composite, etc.)

```
sudo v4l2-ctl --set-input=1
```

And how to change the capture volume:

```
sudo amixer -c Intel set Capture,0 35%,35% unmute cap
```

The standard audio capture rule applies, here: it's better to capture too quiet than too loud. It's impossible to truly
recover from clipping, but sound at 50% of the final level is effectively 15-bit sound, and between a 48kHz capture sample rate,
[10kHz frequency response on the VHS side](https://en.wikipedia.org/wiki/VHS#Audio_recording), and a lossy codec once encoded,
capturing a bit too quiet won't be very noticible.

#### Windows
I've used [Virtualdub2](https://sourceforge.net/projects/vdfiltermod/) in the past, and it works well. If you're
digitizing a lot of tapes, the Linux flow might work better because it's easier to script.

## Encoding
### Formats
As of 2020, these are the fomats I considered:

* DVD (MPEG2/AC3). For a nearly 25-year-old format, DVDs are still widely supported. Practically all Blu-ray players play DVDs,
and both the new Playstation and Xbox consoles will include support.
* webm (VP9/Opus). Webm is much more recent, but it's a web standard, likely to be supported by browsers for at least a decade,
and supported by Android devices, PCs, and Samsung smart TVs. [VP9 has been accelerated on most devices since 2015](https://news.ycombinator.com/item?id=20815322).
* mp4 (x264/aac). [Apple refuses to support VP9](https://discussions.apple.com/thread/250850241). These are their best, widely
suppoted container and codecs.
* mkv (ffv1/flac). There isn't a "standard" archival format, but these lossless codecs and container are all open-source and well-supported.

I ruled out x265 because it isn't supported on older iPhones, and I ruled out AV1 because acceleration isn't widespread, yet, and
it's "expermintal" in ffmpeg.

### Workflow
As I discovered more and more issues with my captured and encoded videos, I put together a scripted pipeline for repeating the
process from source videos.

1. Fix audio timing issues

   When playing back captured videos in VLC, the audio was synced, but after clipping the video with eithe ffmpeg or Virtualdub2,
   the audio would be out-of-sync. [This solved my issue](https://superuser.com/a/1054478):

   ```
   ffmpeg -i $in \
       -f lavfi -i "aevalsrc=0:c=2:s=48000" -filter_complex "[0:a][1:a]amerge[a]" \
    -map 0:v -map "[a]" -ac 2 \
    -acodec flac -vcodec ffv1 $out
   ```

2. Extract clips from source file

   [ffmpeg guide on seeking](https://trac.ffmpeg.org/wiki/Seeking)

   ```
   ffmpeg -i $in -ss ... -to ... -c:v copy -c:a copy -y $out
   ```

   Some videos had multiple segments separated by bits of noise I need to remove.
   This script clips together multiple chunks from a single source video:

   ```
   # Usage: trim <input> <output> <start1> <end1> <start2> <end2> ...
   function trim {
    in=$1
    out=$2
    args=( "$@" )
    filelist=`mktemp --suffix .lst`
    x=2
    while [ $x -lt "$#" ]; do
        file=`mktemp --suffix .mkv`
        echo "file $file" >> $filelist
        ffmpeg -i $in -ss ${args[$x]} -to ${args[$x+1]} -c:v copy -c:a copy -y $file
        let x=x+2
    done

    ffmpeg -f concat -safe 0 -i $filelist -c copy $out
    cat $filelist
    cat $filelist | sed 's/^file //' | xargs rm
    rm $filelist
   }
   ```

3. Normalize the sound
   Use [EBU R 128 loudness normalization](https://en.wikipedia.org/wiki/EBU_R_128). ffmpeg supports this as [loudnorm](https://trac.ffmpeg.org/wiki/AudioVolume).
   Be sure to use the multipass approach:

   ```
   tmp=$(mktemp)
   ffmpeg -i $in -af loudnorm=linear=true:print_format=json -f null -vcodec copy - 2>&1 |
     awk 'match($0, /.*Parsed_loudnorm_0.*/) { while (getline > 0) print;}' > $tmp
   args=$(cat $tmp | jq -r '"measured_I=\(.input_i):measured_LRA=\(.input_lra):measured_TP=\(.input_tp):measured_thresh=\(.input_thresh):offset=\(.target_offset):linear=true:print_format=summary"')
   ffmpeg -i $in -c:v copy -c:a flac -af loudnorm=$args -ar 48000 -sample_fmt s16 -f matroska $out
   ```

4. Crop the video (and possibly fix interlacing)

   This cropping isn't perfect, but it maintains the aspect ratio and removes borders and head-switching noise. I never figured out why, but
   my captured video had the interlaced fields swapped. `il=ls=1:cs=1` will [swap them](https://lists.ffmpeg.org/pipermail/ffmpeg-user/2016-June/032424.html).
   Be sure to swap the fields before cropping, or else you might flip the field order. You can recognize swapped fields by noticing static parts of the video
   that look someone interlaced. You can also recognize them deinterlacing with YADIF, doubling the frame rate, and interpolating. If the image jumps up and down by one pixel every frame, the fields are swapped.

   ```
   ffmpeg -i $in -vf il=ls=1:cs=1,crop=694:462:14:8 -vcodec ffv1 -acodec flac $out
   ```

5. Encode MPEG2

   This is a two-pass encode. The idea behind two-pass encoding is, for a given amount of space (a DVD is 4.7 GB), allocate the space for the best quality.
   You determine the target bitrate `-b:v` by dividing the target size by the clip duration, subtracting the audio bitrate `-b:a`, and subtracting a bit for
   container overhead.

   ```
   ffmpeg -i $in -aspect 4:3 -top 1 -vf scale=352:480:interl=1 \
       -sws_flags lanczos -pass 1 -vcodec mpeg2video \
       -flags +ilme+ildct -mbd rd -trellis 2 -cmp 2 -subcmp 2 -r 29.97 -pix_fmt yuv420p -g 18 \
       -b:v 3500000 -maxrate:v 9000000 -minrate:v 0 -bufsize:v 1835008 -packetsize 2048 \
       -muxrate 10080000 -acodec flac -f rawvideo -y /dev/null
   ffmpeg -i $in -aspect 4:3 -top 1 -vf scale=352:480:interl=1 \
       -sws_flags lanczos -pass 2 -codec:a ac3 -vcodec mpeg2video \
       -flags +ilme+ildct -mbd rd -trellis 2 -cmp 2 -subcmp 2 -r 29.97 -pix_fmt yuv420p -g 18 \
       -b:v 3500000 -maxrate:v 9000000 -minrate:v 0 -bufsize:v 1835008 -packetsize 2048 \
       -muxrate 10080000 -b:a 224000 -ar 48000 -f dvd -y $out
   ```

   * `-aspect 4:3`: DVDs support 4:3 and 16:9. VHS is 4:3.
   * `-top 1`: My videos had the top field first. If you deinterlace the video with the wrong field dominance, you'll see every other frame being ahead or behind.
   * `vf scale=352:480:interl=1 -sws_flags lanczos`: Resize to 352x480. Most NTSC DVDs you buy are 704x480 or 720x480, but 352x480 is also supported. Wikipedia
   claims VHS resolution is approximately 333×480. Use an interlacing-aware resize, and use the lanczos algorithm.
   * `-flags +ilme+ildct -mbd rd -trellis 2 -cmp 2 -subcmp 2`: supposedly they flags improve the quality.
   * `-r 29.97`: Framerate. DVDs only support a few, but NTSC is one of them.
   * `-pix_fmt yuv420p`: Color space
   * `-g 18 -maxrate:v 9000000 -minrate:v 0 -bufsize:v 1835008 -packetsize 2048 -muxrate 10080000`: ffmpeg has a `-dvd` arg, but it doesn't support 352x480.
   These settings come from the [ffmpeg source](https://github.com/FFmpeg/FFmpeg/blob/5633f9a8a221f7511d5ec9b4c57a21c890271ad0/fftools/ffmpeg_opt.c#L2912).
   * `-b:v 3500000`: Average video bitrate
   * `-b:a 224000`: Audio bitrate
   * `-ar 48000`: Audio sample rate
   * `-f dvd`: Output format

6. Create a DVD image

   disc1.xml:
   ```
   <dvdauthor dest="disc1">
       <vmgm />
       <titleset>
           <titles>
               <video format="ntsc" resolution="352x480" />
               <audio lang="EN" content="normal" />
               <pgc><vob file="file1.vob" /></pgc>
               ...
            </titles>
        </titleset>
   </dvdauthor>
   ```

   ```
   VIDEO_FORMAT=NTSC dvdauthor -x disc1.xml
   mkisofs -dvd-video -o disc1.iso disc1
   ```

7. Encode webm files:

   ```
   ffmpeg -i $in -aspect 4:3 -vf 'yadif=1:0,scale=640:480' \
       -sws_flags lanczos -vcodec libvpx-vp9 -pass 1 -b:v 1130000 \
       -speed 1 -row-mt 1 -pix_fmt yuv420p -acodec copy -f rawvideo -y /dev/null
   ffmpeg -i $in -aspect 4:3 -vf 'yadif=1:0,scale=640:480' \
       -sws_flags lanczos -vcodec libvpx-vp9 -pass 2 -b:v 1130000 \
       -speed 1 -row-mt 1 -pix_fmt yuv420p -acodec libopus -b:a 96k -f webm $out
   ```

   The ffmpeg documentation does a good job [explaining different pass and quality settings](https://trac.ffmpeg.org/wiki/Encode/VP9).

   * `-aspect 4:3`: use a 4:3 aspect ratio.
   * `-vf 'yadif=1:0,scale=640:480' -sws_flags lanczos`: Use the yadif deinterlace algorithm with one frame per *field*, i.e. 59.94 fps, top field first, scale to 640x480, and use the lanczos resize algorithm *after* deinterlacing.
   * `-row-mt 1`: enable multithreading

   Unlike the DVD, I opted for square pixels to maximize compatibility.

## Archiving
I chose [MDisc DVD-Rs](https://www.amazon.com/Verbatim-M-Disc-DVD-R-Branded-Surface/dp/B011PZA68Y). They claim they're significantly more durable than standard DVDs and
can last a millenium. For the truly paranoid, get discs from multiple vendors.

The other choice for a physical medium is hard drives. The USB interface has been around for over 20 years and is compatible with USB-C, so expect at least 10 more years of
support. SATA is over 15 years old, and a drive from 2003 works effortlessly with modern hardware.

Avoid flash memory because it can lose data as it ages, especially when untouched.

FAT-32 support is supported almost everywhere, exFAT slightly less so.

## Further reading
* [AV Artifact Atlas](https://bavc.github.io/avaa/index.html)
* [Canadian government guide on the digitization of VHS tapes](https://www.canada.ca/en/conservation-institute/services/conservation-preservation-publications/technical-bulletins/digitization-vhs-video-tapes.html)
