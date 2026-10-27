---
title: Spiko English
key: spiko
locale: ko
hidden: false
featured: true
year: 2025
role: 공동창업자, 테크 리드
stack: [React Native, Expo, Cloudflare Workers, D1, BigQuery]
tagline: 우즈베키스탄 학생을 위한 AI 영어 말하기 시험 연습 앱. 10만+ 다운로드, 4.8점, 흑자 운영.
demo: https://play.google.com/store/apps/details?id=org.milliytechnology.spiko
order: 1
---

## 문제

우즈베키스탄에서 국가 영어 시험을 준비하는 학생들은 피드백이 있는 말하기 연습이 필요하지만,
대부분 과외 비용을 감당하기 어렵습니다. 글로벌 앱은 다른 시장 기준의 가격이고 현지 결제 수단을
지원하지 않습니다.

## 접근

- 답변을 말하면 앱이 녹음하고, 서버에서 AI가 받아쓰고 채점합니다.
- React Native와 Expo, 그리고 직접 만든 OTA 업데이트 배포로 수정 사항을 당일에 전달합니다.
- 저가형 안드로이드 기준 설계: 가능한 기기는 Opus, AAC 인코더가 파일을 깨뜨리는 기기는 WAV,
  파일 크기에 맞춰 늘어나는 업로드 타임아웃.
  [자세한 글](/en/blog/voice-ai-on-cheap-android/).
- Cloudflare Workers와 D1 백엔드, Google Play·App Store 결제와 함께 현지 결제(Payme, 계좌이체).
- 가격과 페이월은 BigQuery의 Firebase 데이터로 실험합니다.

## 성과

- Google Play 10만+ 다운로드, 평점 4.8.
- 안드로이드 세션 98.6% 크래시 없음 (Crashlytics, 2026-09-01 ~ 10-25).
- Opus 기기에서 말하기 답변 업로드 실패율 1.1%.
- 인프라, AI 추론, 플랫폼 수수료, 세금을 제외하고 흑자.
- 커밋 4,300개 이상.

## 배운 점

- 아이디어와 유통은 공동창업자가, 저는 개발과 운영을 맡았습니다. 제품은 팀이 만듭니다.
- 사용자가 실제로 쓰는 기기에서 테스트하세요. 모든 수정은 인코더, 파일 크기, 안드로이드 버전,
  업로드 시간 같은 지루한 로그에서 나왔습니다.
- 만들지 살지는 표준이 아니라 사용자 한 명이 내는 돈으로 정하세요.

[Google Play](https://play.google.com/store/apps/details?id=org.milliytechnology.spiko)와 [App Store](https://apps.apple.com/us/app/spiko-ai-exam-prep/id6763239914)에서 받을 수 있습니다. 업데이트와 지원은 [텔레그램 채널](https://t.me/MilliyTechnology)에서 합니다.
