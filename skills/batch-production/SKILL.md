---
name: batch-production
description: 複数本のゆっくり・ずんだもん動画を1回の呼び出しでまとめて作る（最大20本）。「10本まとめて作って」「シリーズで量産」「毎日1本分をまとめて」と頼まれたときに使う。
---

# まとめて量産する

`create_yukkuri_videos_batch` に最大20件をまとめて渡す。各件は `create_yukkuri_video` と**まったく同じ形**。

作例（ゆっくり霊夢・魔理沙の3本を1回で。確認用プレビュー）:
https://github.com/yukkurigen/yukkurigen-mcp/raw/main/samples/batch.jpg

## 手順

1. 台本を件ごとに書く。**`idempotencyKeyPrefix` を必ず付ける**（各件に `<prefix>:<番号>` が冪等キーとして渡る）
2. `create_yukkuri_videos_batch` → 数秒で `jobId`
3. `get_job` が `succeeded` になったら、`result.results` の各件の `projectId` と `renderId` を `get_render` に渡して完成を待つ

```json
{"idempotencyKeyPrefix":"series-2026-09-26",
 "items":[
  {"title":"朝の白湯のすすめ","script":[{"speaker":"reimu","text":"朝いちばんに白湯を飲む人が増えているのよ。"}]},
  {"title":"傘を長持ちさせるコツ","script":[{"speaker":"marisa","text":"傘って、すぐに骨が折れるよな。"}]}
 ]}
```

## 打ち切られたとき（重要）

- 残高切れ・レート上限を受けた時点で打ち切り、残りは `notAttempted` に入る。**これは「失敗」ではなく「まだ作っていない」**
- 投げ直しは2通りだけ:
  1. **全件を同じ `idempotencyKeyPrefix` で送り直す**（作った件は同じ結果が返り、二重に払わない。その件が終わってから24時間以内）
  2. 打ち切った件と `notAttempted` の分だけを、**新しい** `idempotencyKeyPrefix` の新しいバッチで送る
- **同じ prefix のまま一部だけを送らないこと**（番号がずれて別の件の鍵に当たり、足りない動画が作られない）
- まとめても安くはならない（各件ごとに課金）。レート上限は1分あたり 無料5 / スタンダード10 / プロ20 件

## 知っておくこと

- 最初の1本は `output: "preview"` で見た目を確かめてから、本番の件を流す
- 同時に進められるバッチは2件まで
