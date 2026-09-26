---
name: dialogue-25d
description: ずんだもん・四国めたん・春日部つむぎ・あんこもん（と利用者の自作キャラ）が 2.5D で動きながら向き合って話す掛け合い動画を作る。「ずんだもんとあんこもんの掛け合い」「キャラが動く動画」「2.5D」と頼まれたときに使う。
---

# 2.5D で動く掛け合い動画

東北のキャラ（`zundamon` / `metan` / `tsumugi` / `anko`）と、利用者が取り込んだ自作キャラは、
**何も指定しなくても 2.5D で動く**。呼吸、体と頭の揺れ、顔の向き、瞳、ずんだもんの枝豆やしっぽの揺れが付き、
2人は自動で向き合う。専用のテンプレートや引数は無い。

作例（ずんだもん×あんこもん「ずんだ餅とおはぎ、どっちが最強？」12行）:
https://github.com/yukkurigen/yukkurigen-mcp/raw/main/samples/dialogue-25d.mp4

## 作り方

1. 話者を東北のキャラ（または `list_characters` で `custom: true` の自作キャラ）にする。ゆっくり（霊夢・魔理沙）は 2.5D では動かない
2. 2人の掛け合いにする。聞き手の顔も動くので、**ボケとツッコミ**の形が映える
3. 行ごとに **`emotion`（16種）と `pose`（腕の形）** を書くと、表情と身振りが変わる（→ スキル `expressions-poses`）
4. （任意）盛り上がりの 2〜4行に `cameraMode: "dynamic"`（カメラが話者に寄る）。寄りが要らなければどれか1行に `"cameraMode": null`
5. `create_yukkuri_video` で焼く（最初は `output: "preview"` で確認してよい）

```json
{"title":"ずんだ餅とおはぎ、どっちが最強？","idempotencyKey":"zunda-vs-ohagi-1",
 "script":[
  {"speaker":"zundamon","text":"あんこもん、今日こそ決着をつけるのだ！","emotion":"excited","pose":"point","cameraMode":"dynamic"},
  {"speaker":"anko","text":"決着って、いったい何の話？","emotion":"doubtful","pose":"cross_arms"},
  {"speaker":"zundamon","text":"ずんだ餅とおはぎ、どっちがおいしいかに決まってるのだ！","emotion":"smug","pose":"hip"},
  {"speaker":"anko","text":"そんなの、あんこのおはぎに決まってるでしょ。","emotion":"smug","pose":"point"}
 ]}
```

## 知っておくこと

- 話しているキャラはその行の表情とポーズ。聞き手は、直前に話したときの表情を穏やかにして残し、腕は下ろす
- 同じ一座（東北＋自作キャラ）だけが同じ画面に出られる。ゆっくりと混ぜると `400 cast_mismatch`
- 自作キャラの動きは、部品の位置から首と腰を推し量って付ける。部品が読めないなど 2.5D で描けないときは、平面の立ち絵に戻る（消えはしない）
