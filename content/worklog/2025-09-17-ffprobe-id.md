---
title: identifying videos for transcoding from a bug report
date: 2025-09-17T16:54:00-07:00
type: post
subtitle: stupid jq tricks
categories:
- ffmpeg
- jq
---

we received a ticket about set of videos in a particular collection not streaming correctly from our streaming server (audio, no video/just black). i was curious to see what was going on for this collection. i knew that that the h.264 videos were fine... so what's up?
 
```bash
#!/bin/bash

for file in colloquia; do \
    if ! [[ $file =~ (\.vt|sr)t$ ]] # skip the caption files
    then 
        ffprobe -v quiet -of json -show_format -show_streams $file \
          | jq -rc '{filename: .format.filename, codec: [.streams[] | select(.codec_type == "video")] | .[].codec_tag_string}' \
           >> ~/colloquia.ndjson
    fi
done

jq -r 'select(.codec != "avc1") | [.filename, .codec ] |  @csv' < colloquia.ndjson > colloquia-videos-needing-transcoding.csv
```

