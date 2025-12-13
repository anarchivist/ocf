---
title: "native HLS support and wowza audio streams"
date: "2025-12-12T16:07:23-08:00"
meta: true
categories:
- av
- hls
- wowza
math: false
toc: false
hideDate: false
hideReadTime: true
---

we have been getting a series of reports the last few months of users on Chrome and Edge being unable to play <abbr name="HTTP Live Streaming">HLS</abbr> audio streams that are served up through our Wowza Streaming Engine server. this was particularly baffling as both browsers (because of their common basis on Chromium) [recently rolled out native HLS playback](https://windowsreport.com/chrome-on-desktop-to-finally-support-native-hls-playback/). shortly after seeing this breakage last month we confirmed this was a [bug reported in Chromium](https://issues.chromium.org/issues/454630434). at first we decided to punt on addressing this as it'd eventually be fixed by whenever Chrome updated itself, and we confirmed that we could at least get the player work as intended if the browser was launched with `-disable-features=BuiltInHlsPlayer`. not an ideal scenario.

this time around i dug in a little further. while our AV player (which uses on [mediaelement](https://mediaelementjs.com)) was clearly a bit out of date, it was puzzling. i dug in a bit further and inspected our Wowza configuration files more closely. interestingly enough, there was one Wowza "application" (directory configuration; collection; whatever you want to call it) that seemed to work across all browsers. and why was that? for that single collection, Wowza was [configured to packetize audio streams as MPEG-TS](https://www.wowza.com/docs/how-to-configure-wowza-server-to-stream-audio-only-apple-hls-using-transport-stream). lo and behold - we tested it on the Wowza applications where issues had been reported recently, and it worked!

so, the magic secret was to add this to all the collections' `Application.xml` files in the `<HTTPStreamer><Properties/></HTTPStreamer>` section:

```xml
<Property>
    <Name>cupertinoPacketizeAllStreamsAsTS</Name>
    <Value>true</Value>
    <Type>Boolean</Type>
</Property>
```
