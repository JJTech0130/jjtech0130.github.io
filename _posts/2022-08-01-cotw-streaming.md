---
title: "Camp-of-the-Woods Streaming"
category: "reverse-engineering"
tags: web streaming
---

Camp-of-the-Woods (COTW) is a family resort up in the Adirondacks. This post is about their various streams and videos, and how one might make the best use of them.

### Camp Cam

COTW has a nice "Camp Cam" avalible [here](https://www.cotwny.org/cam.shtml) or [here](https://www.camp-of-the-woods.org/camp-cam), which provides a 24/7 livestream overlooking the lake. However, their web-viewer is annoying, as it cuts out occasionally, and has an annoying "HD Relay" logo in the corner. However, I discovered that the branding is a simple image overlay, and the stream itself is clean.

The m3u8 url (discovered by simply opening the Network tab in DevTools) is `https://b10.hdrelay.com/camera/4c5d3b77-a2f8-4437-aa4c-33462cc8d8da/relay/playlist.m3u8`, which can be opened in VLC!

It can even be opened in a web browser: [see this online player](https://bharadwajpro.github.io/m3u8-player/player/#https://b10.hdrelay.com/camera/4c5d3b77-a2f8-4437-aa4c-33462cc8d8da/relay/playlist.m3u8)!

Now, say you want to re-stream this video to something like YouTube Live or Twitch. It's pretty simple:
```sh
ffmpeg -user_agent 'not ffmpeg' -i https://b10.hdrelay.com/camera/4c5d3b77-a2f8-4437-aa4c-33462cc8d8da/relay/playlist.m3u8 -c copy -f flv rtmp://LIVE_STREAM_INGEST_URL
```

Replace `LIVE_STREAM_INGEST_URL` with your ingest URL, and you're done!

**Note:** For some reason, HDRelay blacklists FFmpeg's default User-Agent. This causes it to return `405 Method Not Allowed`, for some unknown reason. Anyway, all you have to do is use the `-user_agent` flag shown above to change it.
{: .notice--warning}

### Chapel and Seminar videos

The chapel and seminar videos can be found [here](http://www.livestream.cotw365.morleyconsulting.xyz/livestream.php) for the current week. However, one might want to, for example, download the video for offline viewing. This is pretty simple, as it turns out that they are simply embedded Vimeo videos.

The videos are private, which means that you need both the video ID, and the hash. This can be discovered using DevToos (automated script to come later).
You should find a `<video>` element, with a source url of something like `https://player.vimeo.com/video/735188929?h=98bdee7354&amp;`. This can be translated into a regular Vimeo URL: `https://vimeo.com/735188929/98bdee7354`, where you can easily download it.

### Chapel livestream

For the livestream, you can see an iframe with a source of something like `https://vimeo.com/event/136341/embed/41a374a9ec`. This can be turned into a regular URL like so: `https://vimeo.com/event/136341/41a374a9ec`. They seem to re-use the same even for every day, but you can double-check.

Anyway, don't abuse this, it's simply so that you can get a tad bit more value out of these resources. Eventually, I plan to make a simple tool to scrape and save the videos for every day.
