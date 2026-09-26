---
name: your-images
description: 利用者が用意した画像（スクショ・写真・図）を、動画の特定の行で好きな位置・大きさに出す。背景を場面ごとに差し替える。「このスクショを見せながら」「この画像を使って」「背景を変えて」と頼まれたときに使う。
---

# 自分の画像を差し込む・背景を差し替える

作例（ずんだもん・四国めたん。自分のサイトの画像を行ごとに出す）:
https://github.com/yukkurigen/yukkurigen-mcp/raw/main/samples/your-images.mp4

## 行ごとの画像 `materialImageUrl`

その行で画面に出す画像の URL（https）。位置と大きさは下の `materialBox`。**その行だけ**に出る。

```json
{"speaker":"zundamon","text":"たとえば、ゆっくり霊夢と魔理沙の見た目はこれなのだ。",
 "materialImageUrl":"https://example.com/screenshot.png","pose":"point"}
```

- 図解カード（`card`）や写真（`photo`）より優先される
- 画像が出ている間、キャラは少し外側へ寄って画面を空ける

## 位置と大きさ `materialBox`（あなたが決める）

画面に対する %（x/y は左上）。画像は枠に収まる大きさで、枠の上端・左右中央に出る。省略するとテンプレートの素材の枠。

```json
{"speaker":"zundamon","text":"この写真を見るのだ。","materialImageUrl":"https://example.com/a.jpg",
 "materialBox":{"x":35,"y":8,"width":30,"height":35}}
```

- 横長で**顔にかからないのは x が 20〜83%**（左右のキャラの顔はその外側）。目安: 既定 `{x:23,y:5,width:54,height:60}` /
  中央に小さく `{x:35,y:8,width:30,height:35}` / 大きく `{x:20,y:3,width:60,height:68}`
- 値の範囲: x・y は -50〜100、width・height は 0 より大きく 150 まで（外れると 400）
- 縦型: y は 16 以上・y+height は 48 以下（タイトルと顔を避ける）
- 字幕（下の約15%）と話者の顔にかからない所を選ぶ。同じ画像を続けて出す行は同じ値にする
- あとから動かす: `update_lines` に `{"index":3,"materialBox":{...}}`（`null` でテンプレートの枠に戻す）

## 背景 `backgroundImageUrl`

- 動画全体の背景: `create_yukkuri_video` の `backgroundImageUrl`（https の画像か動画）。省略すると室内のイラスト
- 場面転換: その行の `backgroundImageUrl`。**以降の行へ引き継がれる**

## 注意

- **https のみ**。内部アドレスや http は 400（課金なし）で断る
- 画像はレンダーのときに取りに行くので、**ログインが要る URL や期限の短い URL は使わない**
- 利用者の画像の権利（使ってよいか）は利用者に確認する
