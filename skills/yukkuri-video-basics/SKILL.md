---
name: yukkuri-video-basics
description: ゆっくり解説・ずんだもん解説の動画（MP4）を YukkuriGen で1本作る基本。つなぎ方、キャラの選び方、台本の渡し方、完成の受け取り方。「ゆっくり解説で〇〇の動画を作って」「ずんだもんで解説動画を」と頼まれたら最初に使う。
---

# ゆっくり解説・ずんだもん解説の動画を1本作る

台本（話者＋セリフの配列）を YukkuriGen に渡すと、音声・字幕・立ち絵・BGM・効果音を入れた **MP4** が返る。
**台本はあなた（AI）が書く。** YukkuriGen のサーバは AI を呼ばない。

作例（霊夢・魔理沙、AI が書いた台本をそのまま焼いたもの）:
https://github.com/yukkurigen/yukkurigen-mcp/raw/main/samples/yukkuri.mp4

## つなぐ

- MCP: `https://app.yukkurigen.com/api/mcp`。OAuth に対応したクライアント（Claude・ChatGPT のコネクタ、Claude Code のプラグイン）は、許可画面で「許可する」を押すだけ。鍵は要らない
- 鍵で使う場合: 利用者に `https://app.yukkurigen.com/settings/api-keys` で発行してもらい、`Authorization: Bearer <KEY>` を付ける
- 無料プランでも、台本づくり・直し・音声づくり・共有リンク・1クレジットのプレビューまで使える。本番の MP4（5クレジット）は有料プラン

## 手順

1. **キャラを選ぶ**: `list_characters`。1本に出せるのは同じ一座だけ
   - ゆっくり: `reimu`, `marisa`（声は AquesTalk）
   - 東北: `zundamon`, `metan`, `tsumugi`, `anko`（声は VOICEVOX。立ち絵は 2.5D で動く）
   - 利用者が取り込んだ自作キャラ（`custom: true` の `id`）は東北と一緒に出せる
   - 混ぜると `400 cast_mismatch`（課金なし）
2. **台本を書く**: `{ "speaker": "zundamon", "text": "…" }` を並べる。1行40文字前後まで。掛け合いにする
   - 最初の3行で「え、そうなの？」と思わせる引き → 章ごとに1つの話題 → おさらい → 締めの一言
   - 読み間違えそうな語は `reading`（ひらがな）を付ける
3. **まず安く試す**: `create_yukkuri_video` に `output: "preview"`（1クレジット・640×360・20秒）
4. **本番**: `output` を付けない（既定 `mp4`、5クレジット）。**`idempotencyKey` を必ず付ける**（応答を落としても同じ鍵で投げ直せば二重に払わない）
5. **受け取る**: MCP は数秒で `jobId` を返す → `get_job` が `succeeded` になったら `result.renderId` → `get_render` で `done === true` かつ `outputFile` が非空になるまで待つ
6. `outputFile`（MP4 の URL）と、概要欄に貼るクレジット（`credits`）を利用者に渡す。VOICEVOX の声は「VOICEVOX:ずんだもん」の表記が規約の条件なので、クレジットは必ず概要欄に入れてもらう

```json
{"title":"コンビニコーヒーが安い理由","idempotencyKey":"coffee-2026-09-26",
 "script":[
  {"speaker":"zundamon","text":"コンビニのコーヒーって、百円ちょっとで飲めるのだ！"},
  {"speaker":"metan","text":"カフェなら四百円はするのに、どうしてあんなに安いのかしら。"}
 ]}
```

応答には `editorUrl`（画面で台本を開く URL）が入る。利用者が画面で見たいと言ったら渡す。

## よくある失敗

- `402 plan_required` / `insufficient_credits`: 会話を終わらせない。応答の `purchase.priceKey` を `create_checkout` に渡し、返った URL を金額と一緒に伝える
- `409 missing_audio`: `render_mp4` の前に `generate_audio` が要る（`create_yukkuri_video` は自分で作るので不要）
- 演出（図解・表情・2.5D の掛け合い・縦型・量産）は、それぞれのスキルを見る
- 全部の仕様: https://github.com/yukkurigen/yukkurigen-mcp/blob/main/SKILL.md
