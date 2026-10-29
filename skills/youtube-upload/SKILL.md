---
name: youtube-upload
description: YukkuriGen で焼き上がった動画を、利用者の YouTube チャンネルへ投稿する（既定は非公開）。「YouTube に上げて」「投稿まで済ませて」と頼まれたときに使う。
---

# YouTube へ投稿する

作例: なし（利用者のチャンネルに投稿するため）。

## 条件

1. **AI から投稿できるのは、クレジットを使って書き出した MP4 だけ**（`render_mp4` か `create_yukkuri_video` の mp4 で焼いたもの）。そうでないレンダーは 403 `render_not_paid`、焼いている途中は 409 `render_not_ready`。共有リンクしか作っていないなら、先に `render_mp4`（動画1分につき1クレジット）を案内するか、利用者に画面（左のメニューの「動画」）から自分で投稿してもらう（画面からの投稿はこの制限を受けない）
2. **鍵に `youtube:upload` のスコープが要る。** 既定の鍵や OAuth のつなぎ方には付いていない。403 `insufficient_scope` が返ったら、利用者に `https://app.yukkurigen.com/settings/api-keys` で「YouTube への投稿も許可する」を選んで鍵を発行してもらう。コネクタ（OAuth）なら、つなぎ直して許可画面で同じ項目を選んでもらう
3. プロジェクトが**チャンネルに紐づいている**こと（未分類は不可）。作るときに `list_channels` の `id` を `channelId` に渡しておく
4. そのチャンネルで **YouTube 連携が済んでいる**こと（未連携なら 400 で理由が返る）

プランは問わない。クレジットパックだけの無料プランでも、クレジットを使って書き出した MP4 なら投稿できる

## 手順

1. 動画を MP4 に焼き上げる（`render_mp4`。`get_render` で `done` かつ `outputFile` が入るまで待つ）
2. `upload_to_youtube` に `projectId`・`renderId`（完了したレンダー）・`title` を渡す。任意で `description`・`tags`・`privacyStatus`（`private` / `unlisted` / `public`）・`thumbnailUrl`（https）
3. 返った `jobId` を `get_youtube_upload`（`projectId` と `jobId`）で追う。`completed` になると `videoId` が入る

## 必ず守ること

- **公開範囲の既定は `private`**（本人しか見られない）。利用者が公開を望んでいると確かめたときだけ `unlisted` / `public` にする
- 概要欄のクレジット（VOICEVOX の声の表記など）は `upload_to_youtube` が自動で足す。自分で消さない
- 投稿は取り消しにくい。タイトル・説明・公開範囲を利用者に確認してから呼ぶ
- 回数の上限: 1人1日10本・同じ `renderId` は3回まで（失敗は数えない）・失敗も含めた試行は1人1日20回まで。超えると 429。同じ動画を何度も上げ直さない
