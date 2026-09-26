---
name: expressions-poses
description: 立ち絵の表情（16種）・腕のポーズ・カメラの寄り・効果音を行ごとに指定して、セリフに合わせてキャラを演技させる。「表情豊かに」「リアクションを大きく」「演出をつけて」と頼まれたときに使う。
---

# 表情・ポーズ・カメラで演技させる

人気の解説動画は**ほぼ毎行で表情が変わる**。同じ表情が3行以上続くと単調に見える。

作例（ずんだもんの16表情とポーズの一覧）:
https://github.com/yukkurigen/yukkurigen-mcp/raw/main/samples/expressions.jpg

## 表情 `emotion`（全キャラ共通）

`joy` 喜び / `excited` 大喜び / `surprise` 驚き / `shocked` ガーン / `sad` 悲しみ / `crying` 泣き /
`angry` 怒り / `troubled` 困り・焦り / `thinking` 考え中 / `confused` 混乱 / `doubtful` ジト目 /
`smug` ドヤ顔 / `embarrassed` 照れ / `relaxed` ほっこり / `explaining` 説明 / `love` 好き

## ポーズ `pose`（腕の形。東北のキャラと自作キャラ）

`point` 指さし / `wave` 手を挙げる / `cheer` 両手を挙げる / `hip` 腰に手 / `think` あごに手 /
`mouth_cover` 口元に手 / `whisper` ひそひそ / `shh` しーっ / `chest` 胸に手 / `mic` マイク /
`cross_arms` 腕組み（あんこもん）/ `peace` ピース（つむぎ）/ `hold` 手を組む（めたん）/ `normal`

キャラに無いポーズは**黙って無視される**。キャラごとに描けるポーズ（`list_characters` の `poses` でも分かる）:

| キャラ | 描けるポーズ |
|---|---|
| zundamon | normal, point, wave, cheer, hip, think, mouth_cover, whisper, chest, mic |
| metan | normal, point, shh, whisper, mouth_cover, mic, hold |
| tsumugi | normal, point, peace, cheer |
| anko | normal, point, wave, cheer, hip, think, mouth_cover, shh, cross_arms, mic |
| reimu・marisa | なし（腕が無い） |
| 自作キャラ | `list_characters` の `poses`（設定したものだけ） |

## カメラと効果音

- `cameraMode`（**任意。入れなくてよい**）: `"dynamic"` でその行のカメラが話者に寄る。使うなら**2〜4回まで**（全行だと効かない）。`"summary"` は立ち絵を消す、`null` は寄らない
  - 寄りを一切入れたくないときは、どれか1行に `"cameraMode": null` と書く（自動が止まる）。作った後なら `update_lines` で寄っている行を `null` にする
- `effect`: 行の頭の効果音（例 `"ショック1"` `"きらーん1"` `"ひらめく1"` `"チーン1"`）

## 自動と手書き

`emotion`・`pose`・`effect`・`cameraMode` は、**1行も書かなければセリフから自動で付く**
（「うぅ」→泣き、「どうしよう」→困り、「一つ目」→指さし など）。1行でも書いた項目は、書いた指定だけが使われる。
`null` で「書いた」に数えられるのは `cameraMode` と `effect` だけ（`emotion`・`pose` に `null` は書けない。止めたいならどれか1行に値を書く）。
自動で付くのは**作るときの1回だけ**。作った後に `update_lines` で変えた指定が付け直されることはない。
細かく演出したいときは全行に書く。

```json
{"speaker":"metan","text":"えっ、儲からなくてもいいってこと！？","emotion":"shocked","pose":"mouth_cover","cameraMode":"dynamic","effect":"ショック1"}
```
