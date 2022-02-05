---
layout: post
title:  "Logging radio traffic with the AssemblyAI API"
author: "Steve O'Neill"
comments: true
categories: projects
tags:
---

I had the idea that I could use a [Software Defined Radio dongle](https://www.rtl-sdr.com/buy-rtl-sdr-dvb-t-dongles/) to record public voice communications (pollice scanner radio, etc.) and transcribe them to text automatically. It turns out this project was much more practical than I thought - but also more expensive. The API fees are large if you are going with one of the big 3rd-party providers (Google, etc.)

!["PE - Ham Radio MTA_1504" by Metro Transportation Library and Archive is licensed under CC BY-NC-SA 2.0"]({{site.url}}/docs/assets/img/3552011876_0fc17ba3c3.jpeg)
*image source: Metro Transportation Library and Archive - Licensed under CC BY-NC-SA 2.0*

I went with Assembly AI and put $10 in my account, non-refilling. I am inexperienced with using APIs but the AssemblyAI documentation was concise and provided some easy-to-use examples. 

I adapted their quickstart guide and made a python script to:

1. Capture audio from my device using [portaudio](http://www.portaudio.com) and pyaudio (python)
2. Use [SoX](https://linux.die.net/man/1/sox) for removing silence from captures and splitting one large .wav into several smaller .mp3s based on durations of silence greater than 1 second.
3. Send the resulting transcriptions to a prettily-formatted Markdown file for posting to this blog.

My code is available at [this repository.](https://github.com/thefirstcircle/scanner-transcriber)

I was inspired by other implementations discussed [here](https://news.ycombinator.com/item?id=23461607). I am using some of the same tools.

I don't have any kind of radio yet so I just played this [YouTube video](https://www.youtube.com/watch?v=Jhr7i-4jbMI) of a guy talking on his ham.

The results were not perfect but they are surprisingly good compared to what I expected. The code needs some more error handling but it's nice to get the results in a clean .md.

The biggest issue at present is that the "date" next to a quote reflects when the .mp3 was submitted to the API, not when the phrase was actually uttered. Remember that the script is recording long (1-2 minute) .wav segments and splitting them into compressed MP3s all at once.

AssemblyAI  has a real-time streaming API using websockets, but all contents of a stream (including white noise and silence) count down against your balance. That's partially why I'm using SoX to strip white noise and periods of silence over a second. I am also looking into speeding up the submitted recordings by 1.5x to stretch my dollar...

If I keep working on this, I will try to get the date to reflect as accurately as possible (currently it can vary by up to two minutes). Otherwise it doesn't have much use as a "book of record" or public safety accountability tool. I am a beginner with Python so I have a lot of room to improve.

Anyways, here it is. The output from radio_transcripts.py looks like this:

## Radiolog


*2022_02_05-01:05:15_PM*
> “*Just M seven RVF go ahead."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM003.mp3)

*2022_02_05-01:05:57_PM*
> “*Seven IVF. This is K Four BBL. Name here is Brian and I'm in a park in North Georgia, just north of Atlanta, Georgia. Where are you."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM004.mp3)

*2022_02_05-01:06:38_PM*
> “*Roger that. This is November 7, Romeo Victor Fox. I'm just outside of Seattle in a little town called Renton. This is the Bears, the Boeing Amateur Radio Club. A site."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM005.mp3)

*2022_02_05-01:07:20_PM*
> “*"*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM006.mp3)

*2022_02_05-01:08:02_PM*
> “*Oh, that's excellent. Well, I sure appreciate you getting back to me. What type of equipment are you using."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM007.mp3)

*2022_02_05-01:09:25_PM*
> “*Let's see. I've got an icon into an ISO pool."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM009.mp3)

*2022_02_05-01:10:07_PM*
> “*"*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM010.mp3)

*2022_02_05-01:10:48_PM*
> “*Are you at your Qth or are you Mobile today."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM011.mp3)

*2022_02_05-01:11:30_PM*
> “*This is a fixed station in seven hour V. I'm not going through a repeater on Capitol Peak which is near Olympia. Just listening in. There's a lot of the event going on today. We're just kind of listening for their checkpoints over a couple."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM012.mp3)

*2022_02_05-01:12:12_PM*
> “*"*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM013.mp3)

*2022_02_05-01:12:54_PM*
> “*Roger that. Well, I won't tie up the repeater. I hope that's a good event for you today. I'm here in North Georgia. I'm using my handheld beaufang going into a repeater on Sweat Mountain. It's our club repeater for North Fulton and North Fulton Amateur Radio League. And. Yeah, so I appreciate it. Like I said, I'm making a YouTube video, so I hope you don't mind if I put you on my channel. I don't get too many views, so you won't be famous."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM014.mp3)

*2022_02_05-01:13:36_PM*
> “*"*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM015.mp3)

*2022_02_05-01:14:59_PM*
> “*Roger that. That's okay with me and seven RV."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM017.mp3)

*2022_02_05-01:15:41_PM*
> “*"*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM018.mp3)

*2022_02_05-01:16:23_PM*
> “*Alright, seven, threes to you and seven RVF. I really appreciate you coming back to me. Have a great day out west and thanks again. Seven, three."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM019.mp3)

*2022_02_05-01:17:05_PM*
> “*All right, so that was cool. Talk to someone just outside of Seattle, Washington, using my handheld. He was at his house and using a base station going into a repeater close to Olympia, Washington. So great. Contact."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM020.mp3)

*2022_02_05-01:17:47_PM*
> “*So I just had a really nice contact in Southern California. He said he was about 40 miles south of Los Angeles. Unfortunately, I didn't record it, so you don't get to hear that one. But let's try a repeater in Arizona."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM021.mp3)

*2022_02_05-01:18:29_PM*
> “*This is K Four BBL in North Georgia. I'm doing a YouTube demonstration video of Echo Link. I'm connected to local repeater here using my handheld. Anyone in Arizona available for a quick QSO."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM022.mp3)

*2022_02_05-01:19:11_PM*
> “*You."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM023.mp3)

*2022_02_05-01:19:53_PM*
> “*Wow. Thanks for coming back to me. I appreciate it. I didn't get the whole call sign. My call is K Four BBL and the name here is Brian."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM024.mp3)

*2022_02_05-01:20:35_PM*
> “*"*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM025.mp3)

*2022_02_05-01:21:19_PM*
> “*"*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM026.mp3)

*2022_02_05-01:22:01_PM*
> “*Well, two people should have basically talked. Maybe we should song great. Oh, no."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM027.mp3)

*2022_02_05-01:22:43_PM*
> “*You."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM028.mp3)

*2022_02_05-01:24:07_PM*
> “*Roger that. Gary. Again, thanks for coming back to me. I'm in a park just north of Atlanta, Georgia, and like I said, I got my handheld into my local repeater connected to you throughout."*

[audio link](/docs/assets/audio/audio_output_processed_2022_02_05-01:03:05_PM030.mp3)
