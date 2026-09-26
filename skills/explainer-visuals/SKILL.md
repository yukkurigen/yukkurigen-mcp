---
name: explainer-visuals
description: 解説動画に図解カード・写真・章の見出し・強調テロップを入れて、YouTube の人気解説動画のような見た目にする。「わかりやすい解説動画」「図解入りで」「写真も入れて」と頼まれたときに使う。
---

# 図解・写真・章・強調入りの解説動画

**サーバは台本から勝手に作らない。** 台本を書いたあなたが、行ごとに次の項目を書く。書いた行にだけ出る。

作例（ずんだもん・四国めたん「コンビニコーヒーが安い本当の理由」）:
https://github.com/yukkurigen/yukkurigen-mcp/raw/main/samples/zunda.mp4

## 行に書く項目

| 項目 | 何が出るか | 目安 |
|---|---|---|
| `chapter` | 横長の画面左上の章の見出し（14文字以内）。次の章まで出る。縦型では出ない | 8行に1つ。タイトルを冒頭に見せるなら最初の章は2〜3行目 |
| `card` | 図解カード `{title, items, style, span}`。`style`: `list` / `steps` / `point` | 7行に1枚、最大6枚 |
| `photo` | 写真の**英語の**検索語 1〜3個。具体的な物から順に、最後は1語 | 4行に1枚 |
| `emphasis` | 中央の強調テロップ（10文字以内）。`{text, color}` で色（yellow / red / blue / green） | 6行に1つ。素材・図解カードが出ている行では出ない |

- カードの項目は**台本にある事実だけ**を、短い体言止めで
- 写真は Openverse の CC0・パブリックドメインから探す。抽象語（taste, morning 等）は見つからない
- 自分の画像を出したいときは → スキル `your-images`

```json
{"speaker":"zundamon","text":"安くできる理由は三つあるのだ。","chapter":"安さの3つの理由",
 "card":{"title":"安くできる3つの理由","items":["豆をまとめて大量仕入れ","ボタンひとつで自動抽出","お客さんのセルフ式"],"style":"steps","span":4},
 "emotion":"smug","pose":"hip"}
{"speaker":"metan","text":"味は大丈夫なのかしら？","photo":["coffee beans","coffee"],"emotion":"troubled"}
{"speaker":"zundamon","text":"人の手間を減らすのが一番の近道なのだ。","emphasis":"手間を減らす"}
```

## 知っておくこと

- 図解や写真が出ている間、横長ではキャラは少し外側へ寄って画面を空ける
- `title`: 横長は左上に出るが、**章の見出しが1つでもあると左上は章になり、タイトルは最初の章より前の行だけ**。
  縦型は画面の上にずっと出る。`titleTelop: false` でどちらも消える
- カードと写真は、**共有リンクを作るとき（`create_preview_link`）か焼くとき**に画像になり、行の素材に入る（それまでは共有リンクにも出ない）
- どれも作ったあとから `update_lines` で足せる・外せる（`chapter` / `emphasis` / `card` / `photo` / `effect`、外すなら `null`）
- 位置を決めたいときは、その行に `materialBox`（→ スキル `your-images`）。card・photo の続く行にも自動で写る
