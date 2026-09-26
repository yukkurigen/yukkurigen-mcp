---
name: review-and-fix
description: 焼く前に利用者へ共有リンクで見せ、「ここを直して」を受けて指定の行だけ直してから MP4 にする。消費なしで何度でも直せる。「確認してから出したい」「3行目を変えて」「もっと〇〇にして」と言われたときに使う。
---

# 焼く前に見せて、直してから出す

**直すことにお金はかからない。** 課金はレンダーだけ。利用者が納得するまで直してよい。

作例（共有リンクの画面。ログイン無しで、音つき・動く状態で見られる）:
https://github.com/yukkurigen/yukkurigen-mcp/raw/main/samples/review-share.jpg

## 流れ

```
create_yukkuri_video { output:"draft" }   # 消費なし・数秒
generate_audio                            # 消費なし（引数なしで、音声が無い行すべて）
create_preview_link                       # 消費なし。返った URL を利用者に渡す
  （利用者の「ここを直して」）
get_project                               # 今の台本を行番号つきで読む
update_lines                              # 直す行だけ渡す
generate_audio                            # 文を変えた行の音声を作り直す
  （同じ共有リンクが新しい内容を映す。作り直さなくてよい）
render_mp4 { preview:{fromLine, toLine} } # 直した所だけ 1クレジットで確認（任意）
render_mp4                                # 本番 5クレジット
```

## 直し方

- `update_lines` には**変える行だけ**を渡す。渡さなかった行は一切変わらない

```json
{"edits":[{"index":3,"text":"実は理由はもっと単純なのだ","emotion":"smug"},
          {"index":5,"backgroundImageUrl":"https://example.com/bg2.jpg"}]}
```

- `text` / `speaker` / `reading` を変えた行は、その行の音声が消える（応答の `audioInvalidated`）。**`generate_audio` を呼んでから**焼く。音声が無い行があると `render_mp4` は課金せずに `409 missing_audio` で止まる
- **音声を作る前に共有リンクを渡さない**（無音で再生される）
- 台本を丸ごと作り直す（`create_yukkuri_video` をもう一度）は最後の手段。良かった行まで消える

## 知っておくこと

- 共有リンクは既定7日で切れる（最大30日）。渡す相手を間違えたら `revoke: true` で作り直す（前の URL は開けなくなる）
- 共有リンクはブラウザで組み立てて再生しているので、細部は書き出した MP4 と完全には同じでない。最終確認は MP4 で
