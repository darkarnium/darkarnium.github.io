---
layout: post
title: 9447 CTF - Super Turbo Atomic GIF Converter
date: 2015-11-29 00:00:00.000000000 -08:00
author: Peter Adkins
image: 
---
Over the last two days I've been participating in the 9447 CTF with a group ephemerally known as ['Moose 1v1'](https://9447.plumbing/user?id=941). As this was my first participation in any form of CTF, and our team managed to snatch the silver for being the second to solve this particular challenge, I thought I'd write-up the solution that our team came up with.

It's worth noting that we burned a few hours going in entirely the wrong direction, which is discussed below. In the end the sources for the application were released as a hint, but that was a few hours after we'd managed to find the flag.

### Enter the Super Turbo Atomic GIF Converter!

The 'Super Turbo Atomic GIF Converter' was released on day two of this years 9447 CTF. The application targetted in this competition was a very simple one-pager, with the goal being to find a way to fetch the flag from `/home/ctf/flag.txt`.

![My god... It's beautiful.](/assets/article_images/2015/home-with-upload.png)
In line with the CTF comp description, the target application would convert an uploaded GIF into a WEBM file - the result of which would be rendered into an HTML response page as a base64 encoded HTML5 video (via `<video src=data:video/webm;base64,...`).

The result of the above operation was a playable HTML5 video:

![Pretty.](/assets/article_images/2015/gif-upload.png)
Although a very small application, it took us a good number of hours (and a lot of coffee) in order to get a valid solve.

### Content-Types vs Filenames
First up, we attempted to see whether we could determine whether files uploaded were written into a web accessible directory. We had a quick poke around for any obviously named temporary upload directories through URL tampering, but this quickly yielded no results. Although unfruitful, it allowed us to determine that there was an upload file size limit of 1023KB (enforced by TCP RST), and that the application written in Python (using Flask / Werkzeug).

The next thing that we attempted was to upload media files of varying types. One of the first attempts was to upload a JPEG file - with a file suffix of `.jpg` - which was met immediately by an error message. This allowed us to determine that the application was performing at least some sort of type filtering.

![Uh oh.](/assets/article_images/2015/jpg-as-jpg-error.png)
We then intercepted and modified another attempted JPEG upload, this time tampering with the `Content-Type` header from the HTTP POST, as well changing the file suffix to `.gif`. This was much more successful, with the service rendering a single-frame WEBM to the page containing our uploaded JPEG file.

!["Yea, I'm a GIF. It says right here on my file suffix."](/assets/article_images/2015/jpg-as-gif-rendered.png)
After a quick discussion and a few more tests we manged to determine that the service was only checking the file suffix, rather than the `Content-Type` HTTP header.

Next up was to determine how the files were being converted, this was acheived by downloaded the resulting file, and checking it for any meta-data that may have been left by the application responsible for converting / transcoding the files.

A quick `strings` examination of the rendered WEBM file resulted in two telling strings that were presented in all converted files: `Lavf56.40.101WA` and `Lavf56.40.101s`. This, combined with the ability to convert almost any input format to WEBM, was a good indication that the service was most likely using `ffmpeg` to perform conversion.

From here, I was relatively hell-bent on attempting command-injection through input filename - encasing Unix commands in the file name inside of backticks as well as `$()`. Unfortunately, none of these attempts were successful. It was at this point that one of our team members, jdoe, made the suggestion that the use of a filename containing an `ffmpeg` filter (such as `concat:`) might work, but subsequent attempts with this method were also unsuccessful.

From here, we were relatively stumped until jdoe made another suggestion, asking whether `ffpmeg` supports playlist files. I was aware from some previous dealings with `ffmpeg` that it supported HLS streaming through use of M3U8 files, so we thought we'd give it a shot. We knocked together a quick M3U8 playlist containing a non-existant HTTP endpoint as the media segment source, gave the file a suffix of `.gif.m3u8` and uploaded it.

After uploading the file, we were immediately greeted by a corrupt WEBM which was a great sign. A quick look at the logs from the server that we added to the M3U8 file as the segment source showed that there had been a query from a `libavformat` user-agent requesting the same invalid path included in our PoC.

![That's a good sign...](/assets/article_images/2015/m3u8-rfi-log-hit.png)
From here, we knew we had a potential way of nabbing the flag, we just weren't sure how to get the flag into a format that would be rendered by `ffmpeg`. As a result, I ended up getting stuck down the rabbit hole. At one point I was attempting to use M3U8 subtitle support to perform a local file inclusion of the flag, hoping that it would be rendered into the file in same manner. Needless to say, this didn't work.

After a few hours of completely invalid attempts, we switched things up a bit and attempted a local file inclusion directly from the uploaded M3U8... Transcoding a text file into a WEBM via M3U8 playlist. that can't possibly work, can it?

![A wild flag appears!](/assets/article_images/2015/m3u8-lfi-flag.png)
...Apparently it can! With that, we had the flag, were up 180 points and nabbed the silver medal for the second to solve to boot :)

The following M3U8 file was the used as our final solution, which simply uses a local absolute file path as the source:

```xml
#EXTM3U
#EXT-X-TARGETDURATION:1
#EXTINF:1,
/home/ctf/flag.txt
#EXT-X-ENDLIST
```

Finally, a big thanks to @9447CTF for running the CTF! :)
