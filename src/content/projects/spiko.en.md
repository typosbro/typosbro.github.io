---
title: Spiko English
key: spiko
locale: en
hidden: false
featured: true
year: 2025
role: Co-founder, tech lead
stack: [React Native, Expo, Cloudflare Workers, D1, BigQuery]
tagline: AI speaking-exam practice for students in Uzbekistan. 100K+ installs, 4.8 stars, cash-positive.
demo: https://play.google.com/store/apps/details?id=org.milliytechnology.spiko
order: 1
---

## Problem

Students in Uzbekistan preparing for the national English exam need speaking
practice with feedback, and most can't afford a tutor for it. Global apps are
priced for other markets and don't take local payment methods.

## Approach

- Speak an answer, the app records it, and AI transcribes and grades it on the server.
- React Native and Expo, with my own over-the-air update delivery, so a fix
  reaches users the same day.
- Built for budget Androids: Opus on capable phones, WAV on phones whose AAC
  encoder corrupts files, upload timeouts that scale with file size.
  [Write-up](/en/blog/voice-ai-on-cheap-android/).
- Backend on Cloudflare Workers and D1; local payments (Payme, bank transfer)
  alongside Google Play and App Store billing.
- Pricing and paywall run as experiments read from Firebase data in BigQuery.

## Outcome

- 100,000+ installs and 4.8 stars on Google Play.
- 98.6% crash-free Android sessions (Crashlytics, Sep 1 – Oct 25, 2026).
- 1.1% upload failure rate for speaking answers on Opus phones (Sep 1 – Oct 25, 2026).
- Cash-positive after infrastructure, AI inference, platform fees and taxes.
- 4,300+ commits.

## What I learned

- The idea and distribution came from my co-founder; my part was building and
  running it. Products are won by teams.
- Test on the phones your users have. Every fix here came from logging boring
  fields: encoder, file size, Android version, upload time.
- Choose build or buy by what a user pays you, not by what's standard.

Available on [Google Play](https://play.google.com/store/apps/details?id=org.milliytechnology.spiko) and the [App Store](https://apps.apple.com/us/app/spiko-ai-exam-prep/id6763239914). Product updates and support run through the [Telegram channel](https://t.me/MilliyTechnology).
