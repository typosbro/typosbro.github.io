---
title: Shipping voice AI to cheap Android phones
key: voice-ai-on-cheap-android
locale: en
date: 2026-10-27
draft: false
tags: [android, audio, engineering, spiko]
excerpt: The demo worked on my phone. Then it met Uzbekistan's real phones — old Androids, software encoders, slow mobile data — and the hard part turned out to be getting a recording off the device intact.
---

Spiko is a speaking-exam practice app for students in Uzbekistan. You answer a question out loud, the app records you, and an AI grades the answer. On my phone, the whole loop took a few seconds and never failed.

My phone is not the market.

## Who actually holds the phone

Over the last eight weeks (September 1 to October 25, 2026), 93% of Spiko's users were on Android. The top brands were Samsung (38%), Xiaomi (32%), Honor (10%), Infinix (6%) and Tecno (5%). About one user in six runs Android 11 or older.

Most of these are not flagships. They are budget phones with little free storage, older system codecs, and mobile data that drops in and out between home and school.

Every assumption I had came from a phone that is none of those things.

## First mistake: running the model on the phone

The first version did speech recognition on the device. It looked like the right call: no server cost, works offline, private by default.

It didn't survive contact with the users. Models in the hundreds of megabytes would not load on phones with little memory and storage, and the ones that did load were slow. The users who needed offline the most were the ones whose phones couldn't run it.

So transcription and grading moved to the backend. The phone's job shrank to one thing: record a clean file and get it to the server.

That sounds easy. It wasn't.

## Second mistake: trusting the platform encoder

Android gives you an AAC encoder and a muxer, and on most phones they just work. On some they don't.

Older budget devices whose only AAC encoder is the legacy software one (`OMX.google.aac.encoder`) produced structurally corrupt `.m4a` files most of the time. In production data from July 1 to August 11, 78 of 137 recordings from that encoder were broken, about 57%. Hardware encoders produced 0 broken files out of 4,525 in the same window.

A corrupt file is the worst kind of failure. The recording "works", the upload "works", and the student gets a grade for silence.

The fix was a device split. Phones that can't be trusted with AAC record WAV instead. WAV is uncompressed and boring, and it had zero corruption failures.

## Third mistake: WAV is huge

Fixing corruption created the next problem. A 120-second answer is about 3.84 MB as WAV, 0.96 MB as AAC, and about 0.36 MB as Opus.

On slow mobile data, that difference is everything. In one week in September, WAV carried 21,903 of 25,479 Android uploads, and 13% of Android uploads took longer than 20 seconds, against 6% on iOS, which sends compressed audio. Same backend, same grading. The only difference was file size.

So capable Androids (Android 11 and up) moved to Opus in an Ogg container at 16 kHz and 24 kbps. That bitrate sounds low, but it is plenty for speech, and it makes the file roughly ten times smaller than the WAV it replaces. Older phones stay on WAV, with an upload timeout that scales with file size instead of a fixed one.

Before writing any of this, I had the agents read how Telegram's Android client records and sends voice messages. It has dealt with every class of cheap Android for over a decade, and voice notes are exactly this problem: speech, small files, bad networks. Copying a proven approach beat inventing one.

## What the numbers look like now

From September 1 to October 25, upload failure rates by format were:

| Format | Upload starts | Failed |
|---|---|---|
| Ogg/Opus | 125,004 | 1.1% |
| WAV | 153,323 | 5.3% |
| AAC | 35,919 | 7.1% |

I want to be careful with this table. It doesn't prove Opus is five times more reliable. The format tracks the phone: newer devices get Opus, the oldest get WAV, and older phones also tend to come with worse storage and worse connections. Part of that gap is the device, not the codec.

What I can say is that the phones on Opus send small files and almost always get through, and the remaining failures sit on the oldest devices. That tells me where to look next.

There's also a gap I can't fill. My earlier telemetry lived in a self-hosted ClickHouse on a machine whose SSD died, so I don't have a clean before-and-after for the switch. If you run your own analytics, back them up.

## Build or buy, priced in so'm

The other half of this story is cost. My users pay a few dollars a month, when they pay at all. Tools priced for US apps don't fit that.

So the choices went by unit economics, not by what's standard:

- **React Native**, because over-the-air updates mean a fix reaches broken phones the same day, without waiting for users to update from the store.
- **My own OTA distribution** instead of a paid hosted service, because per-user pricing didn't make sense at this revenue per user.
- **Self-hosted analytics** instead of a hosted product analytics tool, for the same reason. That one cost me history when the disk died, which is the honest price of that decision.
- **My own payments layer** instead of a subscription platform, because local payment methods aren't covered by the usual tools anyway.

None of these is right in general. Each was right for this market.

## What I'd tell myself at the start

1. Test on the phone your users have, not the one you have. A used budget Android costs less than one day of debugging blind.
2. Log the boring fields: encoder name, file size, Android version, upload duration. Every fix in this post came from one of them.
3. A corrupt file is worse than a failed upload. Fail loudly.
4. When a format table looks too good, check what else the format is standing in for.

The AI part of an AI app is often the easy part. The work is getting a clean, small recording from a cheap phone on a bad connection to a server, hundreds of thousands of times.
