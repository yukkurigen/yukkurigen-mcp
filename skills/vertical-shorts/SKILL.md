---
name: vertical-shorts
description: YouTube ショート・TikTok 向けの縦型（9:16）ゆっくり・ずんだもん動画を作る。「ショート動画」「縦型で」「TikTok 用」「1分以内」と頼まれたときに使う。
---

# 縦型ショート（9:16）

`create_yukkuri_video` に **`platform: "shorts"`**（または `"tiktok"`）を渡すだけで縦型になる。既定は `youtube`（横型）。
向きは**プロジェクトに残る**ので、draft のあとや直したあとの `render_mp4` は `platform` を省いても縦型で焼かれる
（`get_project` の `platform` で確かめられる）。1クレジットのプレビューと共有リンクも縦型で描かれる。

作例（ずんだもん・四国めたん「ペットボトルを満タンで凍らせない理由」8行）:
https://github.com/yukkurigen/yukkurigen-mcp/raw/main/samples/vertical-shorts.mp4

## 台本のコツ（ショート向け）

- **1行目で結論か驚き**（「〇〇はダメなのだ！」）。スクロールで流される前に止める
- 全体で 6〜10行・1分以内。1行は短く（縦型は字幕の幅が狭い）
- 数字や要点は `emphasis`（強調テロップ）で画面の真ん中に大きく
- 最後は一言で締める（「覚えておくわ」「チャンネル登録してね」）

```json
{"title":"ペットボトルを満タンで凍らせない理由","platform":"shorts","idempotencyKey":"shorts-bottle-1",
 "script":[
  {"speaker":"zundamon","text":"ペットボトルの水を凍らせるとき、満タンはダメなのだ！","emotion":"excited","pose":"point"},
  {"speaker":"metan","text":"えっ、どうして？","emotion":"surprise"},
  {"speaker":"zundamon","text":"水は凍ると、体積がおよそ1割ふえるのだ。","emotion":"explaining","emphasis":"約1割ふえる"}
 ]}
```

## 縦型の画面

上から **タイトル（画面の上 約15%）→ 素材（16〜46% あたり）→ キャラ2人を左右に大きく → 字幕（74〜90%）**。
字幕は横長より大きく出る。素材を行ごとに置くなら `materialBox` の y は 16 以上・y+height は 48 以下が目安。
配置そのものを直すときは `update_template_layout` に `"aspect":"9:16"`（今の値は `list_templates` の `portraitLayout`）。

## 知っておくこと

- 同じ台本を横型と縦型の両方で出したいときは、横型で作ったあと `render_mp4` に `platform: "shorts"` を付けてもう一度焼く（もう5クレジット）
- 立ち絵・表情・2.5D の動きは横型と同じように効く
