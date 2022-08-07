---
title: "Downloading Vimeo videos"
category: "reverse-engineering"
tags: web download video
---

So, if you saw my previous blog post, you know about how I wanted to download chapel videos from COTW.
This is a continuation post on how I automated that. This mainly consists of using Vimeo's API to download the videos.

Here's the final code I ended up using:

```bash
# extract the URLs
URLS=$(curl -s 'http://www.livestream.cotw365.morleyconsulting.xyz/chapels.php' | pup "iframe attr{src}")

# yes, the XMLHttpRequest part is important for some reason...
json=$(curl -s 'https://vimeo.com/_next/jwt' -H 'X-Requested-With: XMLHttpRequest')
JWT=$(echo $json | jq '.token' --raw-output)

# For each video URL...
for URL in $URLS
do
    # keep everything after the last slasg
    trimmed=$(echo $URL | sed 's:.*/::')
    # everything before ?h= is the ID
    ID=${trimmed%?h=*}
    # everything after ?h= is the hash
    temp=${trimmed##*?h=}
    # throw away everything after &, it's garbage
    HASH=${temp%%&*}

    # print debug info
    echo ""
    echo "Trying to download video: $ID"
    echo "With hash: $HASH"
    echo "and token: $JWT"
    echo ""

    # get info about formats
    json=$(curl -s "https://api.vimeo.com/videos/$ID:$HASH?fields=download.height%2Cdownload.link&action=load_download_config" -H "Authorization: jwt ${JWT}")
    # hardcoded 1080p
    DOWNLOAD=$(echo $json | jq '.download[] | select(.height == 1080) | .link' --raw-output)

    wget $DOWNLOAD
done
```

1. I extracted the video URLs from the original page, using `pup`, simply selecting the `src` attribute of all iframes

2. I got a JWT token for Vimeo's API

3. I trimmed out the video ID and hash from the original (embed) URL

4. I asked Vimeo's API for download URLs using the above info

5. I parsed the response with `jq`, filtering for the 1080p video

6. I downloaded the video with `wget`

This is *much* faster than `yt-dlp` for some reason, usually getting like 20mb/s.

This is just a basic "I did it" post, as I don't really know how to elaborate :smile:...
