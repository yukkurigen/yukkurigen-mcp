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
check_readings                            # 消費なし。エンジンが実際に読む音を見て、読み違いをルビ・辞書で直す
generate_audio                            # 消費なし（引数なしで、音声が無い行・読みが変わった行すべて）
create_preview_link                       # 消費なし。返った URL を利用者に渡す
  （利用者の「ここを直して」）
get_project                               # 今の台本を行番号つきで読む
update_lines                              # 直す行だけ渡す（行数は変わらない）
insert_lines                              # 行を追加したいとき（行番号が再割り当てされる）
split_line                                # 1行を複数に分けたいとき
delete_lines                              # 行を消したいとき
check_readings                            # 直した行の読みを確かめる
generate_audio                            # 文を変えた行・新規行・読みが変わった行の音声を作る
  （同じ共有リンクが新しい内容を映す。作り直さなくてよい）
render_mp4                                # 本番（動画1分につき1クレジット）
```

## 直し方

- `update_lines` には**変える行だけ**を渡す。渡さなかった行は一切変わらない。行数を変えたいときは `insert_lines` / `split_line` / `delete_lines` を使う（いずれも行番号が再割り当てされる。終わったら `get_project` で最新の行番号を確認する）

```json
{"edits":[{"index":3,"text":"実は理由はもっと単純なのだ","emotion":"smug"},
          {"index":5,"backgroundImageUrl":"https://example.com/bg2.jpg"}]}
```

- `text` / `speaker` / `reading` / `voice`（その行の声の速さ・高さ）を変えた行は、その行の音声が消える（応答の `audioInvalidated`）。**`generate_audio` を呼んでから**焼く。音声が無い行があると `render_mp4` は課金せずに `409 missing_audio` で止まる
- **読み違い**（「今日」をこんにち、人名など）は `check_readings` の `kana` で見つける。その行のその語だけなら `text` にルビ `｜今日《きょう》`（字幕は漢字のまま、声だけ読みになる）、何度も出る語なら `set_pronunciation` で辞書に入れる。辞書を変えた行は `get_project` の `staleAudio` に出るので、`generate_audio`（引数なし）で作り直す
- 「ここだけ早口に」「もっとゆっくり」はその行の `voice`（`{"speed":1.3}`・`{"speed":0.8,"pitch":-0.04}`。speed 0.5〜2.0・pitch -0.15〜+0.15）。動画全体の速さは `update_project_settings` の `voicePlaybackRate`（既定 1.1。音声の作り直しは要らず、共有リンクですぐ確かめられる）
- **音声を作る前に共有リンクを渡さない**（無音で再生される）
- 台本を丸ごと作り直す（`create_yukkuri_video` をもう一度）は最後の手段。良かった行まで消える

## 知っておくこと

- 共有リンクは既定7日で切れる（最大30日）。渡す相手を間違えたら `revoke: true` で作り直す（前の URL は開けなくなる）
- 共有リンクはブラウザで組み立てて再生しているので、細部は書き出した MP4 と完全には同じでない。最終確認は MP4 で
