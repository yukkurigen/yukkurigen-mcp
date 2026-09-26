---
name: youtube-upload
description: YukkuriGen で焼き上がった動画を、利用者の YouTube チャンネルへ投稿する（既定は非公開）。「YouTube に上げて」「投稿まで済ませて」と頼まれたときに使う。
---

# YouTube へ投稿する

作例: なし（利用者のチャンネルに投稿するため）。

## 条件（3つ）

1. **鍵に `youtube:upload` のスコープが要る。** 既定の鍵や OAuth のつなぎ方には付いていない。403 が返ったら、利用者に `https://app.yukkurigen.com/settings/api-keys` でこのスコープ付きの鍵を発行してもらう
2. プロジェクトが**チャンネルに紐づいている**こと（未分類は不可）。作るときに `list_channels` の `id` を `channelId` に渡しておく
3. そのチャンネルで **YouTube 連携が済んでいる**こと（未連携なら 400 で理由が返る）

## 手順

1. 動画を焼き上げる（`get_render` が完成を返すまで）
2. `upload_to_youtube` に `projectId`・`renderId`（完了したレンダー）・`title` を渡す。任意で `description`・`tags`・`privacyStatus`（`private` / `unlisted` / `public`）・`thumbnailUrl`（https）
3. 返った `jobId` を `get_youtube_upload`（`projectId` と `jobId`）で追う。`completed` になると `videoId` が入る

## 必ず守ること

- **公開範囲の既定は `private`**（本人しか見られない）。利用者が公開を望んでいると確かめたときだけ `unlisted` / `public` にする
- 概要欄のクレジット（VOICEVOX の声の表記など）は `upload_to_youtube` が自動で足す。自分で消さない
- 投稿は取り消しにくい。タイトル・説明・公開範囲を利用者に確認してから呼ぶ
