# YukkuriGen — ゆっくり動画を作る (Agent Skill)

テーマから「ゆっくり解説/劇場」動画の **MP4** を生成するためのスキル。
あなた（AI）が台本を書き、YukkuriGen が音声を合成してレンダーし、動画のURLを返します。
ユーザーはそれをそのまま投稿できます。

## 接続

- 発行: ユーザーに `https://app.yukkurigen.com/settings/api-keys` でAPIキーを発行してもらう（コネクタや VS Code から OAuth でつなぐなら不要）。
  **無料プランでもここまで使える**（すべて0クレジット）:
  - 台本の作成・読み返し・修正（`create_yukkuri_video` の `output:"draft"`、
    `get_project`、`update_lines`、`generate_audio`）
  - **共有リンク**（`create_preview_link`）——全編を音つきで、ログイン無しに見られる。
    まず動くものを見せたいときはこれを使う。
  - 本番の MP4 レンダー（全編・本番解像度・**動画1分につき1クレジット**、端数切り上げ）だけはクレジットが要る。
    無料プランは毎月の付与が0クレジットなので、**クレジットパック（買い切り・期限なし。無料プランのままでも買える）**か、
    有料プラン〈スタンダード以上〉または買い切りライセンスが必要。どれも無ければ `402 plan_required` が返る
    ——そのときは下の「支払いが要るとき」に従って購入まで案内する。
- MCP（ChatGPT・claude.ai などのコネクタ）: `https://app.yukkurigen.com/api/mcp` を追加するだけ。
  OAuth 2.1（動的クライアント登録・PKCE）で YukkuriGen の許可画面が開き、利用者が「許可する」を押すと
  つながる。**鍵の発行は要らない。** 発行されたトークンは利用者の鍵一覧に「<アプリ名>（OAuth）」として出て、
  そこから取り消せる。スコープは画面で発行する鍵と同じ既定（`youtube:upload` は、許可画面で利用者が
  「YouTube への投稿も許可する」を選んだときだけ付く。クライアントが `scope` で求めても、それだけでは付かない）。
- MCP（VS Code）: MCP サーバーを種類 `http`・URL `https://app.yukkurigen.com/api/mcp` で追加するだけ。
  VS Code が認証してよいかを聞くので許可し、ブラウザで開く YukkuriGen の許可画面で「許可する」を押すと
  つながる。**鍵の発行は要らない**（2026-10-05 に VS Code 1.139 で、ツール一覧が出るまで確認）。
  鍵一覧には「Visual Studio Code」に「OAuth で接続」の札が付いて出る。
- MCP（Cursor・Windsurf など、ほかのコードエディタ）: 同じ URL でつながるか試してよい（**実機では未確認**）。
  つながらなければ、次のヘッダーで鍵を渡す方法を使う。
- MCP（ヘッダーを渡せるクライアント。Claude Code・Codex CLI・Cursor など）: `claude mcp add --transport http yukkurigen https://app.yukkurigen.com/api/mcp --header "Authorization: Bearer <KEY>"`
- REST: `Authorization: Bearer <KEY>` を付けて `https://app.yukkurigen.com/api/v1/...` を叩く。

## ツール（MCP）/ エンドポイント（REST）

| やること | MCP tool | REST |
|---|---|---|
| キャラ一覧 | `list_characters` | `GET /api/v1/characters` |
| 残高確認 | `get_credits` | `GET /api/v1/credits` |
| 台本→MP4 を一括生成 | `create_yukkuri_video` | `POST /api/v1/agent/generate` |
| 複数本をまとめて作る（最大20） | `create_yukkuri_videos_batch` | `POST /api/v1/agent/generate/batch` |
| プロジェクト一覧 | `list_projects` | `GET /api/v1/projects` |
| 台本を読み返す（行番号つき） | `get_project` | `GET /api/v1/projects/{id}` |
| 指定した行だけ直す | `update_lines` | `PATCH /api/v1/projects/{id}/lines` |
| 行を挿入する | `insert_lines` | `POST /api/v1/projects/{id}/lines/insert` |
| 行を分割する | `split_line` | `POST /api/v1/projects/{id}/lines/split` |
| 行を削除する | `delete_lines` | `POST /api/v1/projects/{id}/lines/delete` |
| **読みを確かめる（音声の前に必ず）** | `check_readings` | `POST /api/v1/projects/{id}/readings/check` |
| 音声を作る（MP4の前に必須） | `generate_audio` | `POST /api/v1/projects/{id}/audio/generate` |
| 読みの辞書に登録する（語→読み） | `set_pronunciation` | `PUT /api/v1/pronunciations` |
| 読みの辞書の一覧 | `list_pronunciations` | `GET /api/v1/pronunciations` |
| 読みの辞書から消す | `delete_pronunciation` | `DELETE /api/v1/pronunciations`（クエリ `word` に消す語） |
| 非同期ジョブの状態取得 | `get_job` | `GET /api/v1/jobs/{jobId}` |
| **焼く前に人へ見せる共有リンク** | `create_preview_link` | `POST /api/v1/projects/{id}/preview-link` |
| YouTube へ投稿する | `upload_to_youtube` | `POST /api/v1/projects/{id}/upload` |
| 投稿ジョブの状態 | `get_youtube_upload` | `GET /api/v1/projects/{id}/upload/{jobId}` |
| 上げた動画の実績（動画ごと） | `get_channel_video_analytics` | `GET /api/v1/channels/{channelId}/analytics/videos` |
| 1本の視聴維持率の推移 | `get_video_retention` | `GET /api/v1/channels/{channelId}/analytics/videos/{videoId}/retention` |

| 見た目の一覧 | `list_templates` | `GET /api/v1/agent/templates` |
| 見た目を複製する（直す前に） | `copy_template` | `POST /api/v1/agent/templates/copy` |
| キャラ・字幕・素材の位置を直す | `update_template_layout` | `PATCH /api/v1/agent/templates/{templateId}/layout` |
| 字幕の色・大きさ・フチ、BGM の音量・ダッキング、背景などの見た目を直す | `update_template_style` | `PATCH /api/v1/agent/templates/{templateId}/style` |
| 作った動画の見た目を差し替える | `set_project_template` | `PUT /api/v1/projects/{id}/template` |
| 動画全体の読み上げの速さ・BGM のダッキング・声だけのキャラ（voiceOnly）を焼かずに変える | `update_project_settings` | `PATCH /api/v1/projects/{id}/settings` |
| チャンネルの一覧 | `list_channels` | `GET /api/v1/channels` |
| 既存プロジェクトを MP4 に | `render_mp4` | `POST /api/v1/projects/{id}/render` |
| レンダー進捗/出力URL | `get_render` | `GET /api/v1/projects/{id}/render/{renderId}/progress` |
| 決済リンクを出す（402のとき） | `create_checkout` | `POST /api/v1/billing/checkout-link` |
| 支払いが済んだか確認 | `get_checkout` | `GET /api/v1/billing/checkout-link/{sessionId}` |
| 画像を取り込む（期限付きURLを永続化） | `import_images` | `POST /api/v1/assets/import` |
| 手元の画像を上げる URL を出す（シェルがあるとき） | `create_image_upload` | `POST /api/v1/assets/image-uploads` |
| PUT した画像を確かめて URL を得る | `confirm_image_upload` | `POST /api/v1/assets/image-uploads/confirm` |
| 手元の画像を base64 で上げる（シェルが無いとき） | `upload_image` | `POST /api/v1/assets/image-uploads/inline` |
| 行の BGM・効果音に書ける音の一覧（組み込み・自分が上げた音） | `list_bgm` | `GET /api/v1/audio-library` |
| 手元の音（BGM・効果音）を上げる URL を出す（シェルがあるとき） | `create_audio_upload` | `POST /api/v1/assets/audio-uploads` |
| PUT した音を確かめて URL を得る | `confirm_audio_upload` | `POST /api/v1/assets/audio-uploads/confirm` |
| 手元の音を base64 で上げる（シェルが無いとき） | `upload_audio` | `POST /api/v1/assets/audio-uploads/inline` |
| AI（ACE-Step）で BGM を1曲作って自分の BGM 素材にする（2クレジット・非同期） | `generate_bgm` | `POST /api/v1/assets/bgm/generate` |
| 自分のキャラに選べる声の一覧 | `list_voices` | `GET /api/v1/voices` |
| 上げた画像から自分のキャラを作る | `create_character` | `POST /api/v1/agent/characters` |
| 自分のキャラの名前・声・表情ごとの画像・顔の位置・大きさを直す | `update_character` | `PATCH /api/v1/agent/characters/{characterId}` |

`list_projects` / `get_project` / `update_lines` / `insert_lines` /
`split_line` / `delete_lines` / `generate_audio` / `check_readings` /
`set_pronunciation` / `list_pronunciations` / `delete_pronunciation` /
`create_preview_link` / `import_images` / `create_image_upload` / `confirm_image_upload` /
`upload_image` / `list_voices` / `create_character` / `update_character` / `update_project_settings` は**消費なし**。
焼く前の確認は共有リンク（`create_preview_link`）だけ。低解像度の MP4 プレビュー（`output:"preview"` /
`render_mp4` の `preview`）は 2026-09-27 に廃止した（`render_mp4` の `preview` は 410 `preview_removed`、`output:"preview"` は 400 `validation_error`。どちらも課金なし）。
本番の MP4 は**動画の長さで決まる**: 1分につき1クレジット（端数は切り上げ。30秒でも1、2分10秒なら3）。
BGM の生成（`generate_bgm` / `POST /api/v1/assets/bgm/generate`・`POST /api/v1/projects/{projectId}/bgm/generate`）は1曲2。
課金は MP4 の書き出しと BGM 生成で起きる。直すこと自体にお金がかからないので、ユーザーが
納得するまで何度でも直してよい。

**共有リンクを渡したら、MP4 にする道も一緒に伝える。** `create_preview_link` の返事（REST は応答の `mp4`）には
次の一手が付く: MP4 にしたときのクレジットの**見積もり**（台本の文字数から多めに出す。実際の額は書き出しを
始めるときの長さで決まる）・いまの残高とプラン（API キーに `credits:read` があるとき）・`next`
（足りれば `render_mp4`、足りなければ `create_checkout` と `purchase.priceKey`＝不足を満たすいちばん小さい
クレジットパック。無料プランなら月額の `alternative`（スタンダード）も付く。残高を見られない鍵なら `get_credits`）。
無料プランのままでもクレジットパックは買え、残っている間は MP4 を書き出せる。持ち主がログインした状態で
共有リンクを開くと、ページの「MP4 に書き出す」ボタンからも書き出せる（見る人には出ない）。

### 焼く前に、人に見てもらう（消費なし）

**MP4 を焼く前に共有リンクを使うこと。** ログイン不要で開けるURLが返るので、
そのまま利用者へ渡す。相手はブラウザで**音つき・動く状態**を確認できる
（動画ファイルは出ない）。

    create_yukkuri_video { output:"draft" }   # 消費なし
    check_readings                            # 消費なし。読み違いをルビ・辞書で直す（下の「読みを確かめて直す」）
    generate_audio                            # 消費なし
    （共有リンクを発行して渡す）                # 消費なし
    （相手の OK を待つ。直したら update_lines → check_readings → generate_audio。
      **同じURLがそのまま新しい内容を映す**ので、作り直さなくてよい）
    render_mp4                                # 動画1分につき1クレジット

**音声を作る前にリンクを渡さないこと**——無音で再生され、確認にならない。

`generate_audio` は**引数なしで呼べる**（音声が無い行と、辞書の変更で読みが変わった行〈`get_project` の `staleAudio`〉すべてが対象）。REST を
直接叩くときも本文は空 `{}` でよい。特定の行だけ作り直したいときは、
**MCP なら `lineIndexes`（行番号）**、**REST なら `lineIds`（REST の `GET /projects/{id}` が返す `lines[].id`）**に
並べて `force: true` を付ける（MCP の `get_project` は行の id を返さない。行番号で指定する）。

### 非同期ジョブ（Prefer: respond-async）

`generate_audio`（POST /audio/generate）、
BGM 生成（POST /bgm/generate）、BGM パイプライン（POST /bgm/pipeline）は、
`Prefer: respond-async` ヘッダを付けて呼ぶとサーバー側で非同期ジョブになる
（サーバの設定によっては付けてもジョブにならず、同期で処理して 200 を返す）。
`generate_bgm`（POST /api/v1/assets/bgm/generate）は**いつもジョブ**で、`Prefer` が無くても 202 を返す
（ジョブが使えない設定のサーバでは 503 `jobs_unavailable`。同期では受けない）。

```
POST /api/v1/projects/{id}/audio/generate
Prefer: respond-async
```

レスポンス（202）:
```json
{ "jobId": "...", "status": "queued", "pollUrl": "/api/v1/jobs/{jobId}" }
```
Header: `Preference-Applied: respond-async`

`get_job` / `GET /api/v1/jobs/{jobId}` でポーリングする:
- `status: "queued"` / `"running"` → まだ処理中
- `status: "succeeded"` → `result` に完了データ（同期パスの 200 ボディと同じ形）
- `status: "failed"` → `error.code` に理由（`error.message` に説明、`error.httpStatus` も入る）

**REST から直接呼ぶときは `Prefer: respond-async` と `Idempotency-Key` ヘッダを付けること。**
`Prefer` を付けない呼び出しは同期のまま。ただしサーバの設定によっては、同じ受け付けのあと
一定時間だけ完了を待ち、終われば同期と同じ 200（失敗なら同期と同じ形のエラー）、終わらなければ
`Preference-Applied` なしの 202 と `jobId` を返す。**202 が返りうるものとして扱うこと。**
同期で処理されたときは `Idempotency-Key` が効かず、台本・BGM は投げ直した分だけ課金される。

MCP では `generate_audio` と `create_yukkuri_video`（mp4）と
`create_yukkuri_videos_batch`（下の「大量に作る」）のツールが自動的に
この非同期パスを使い、jobId を返す。`get_job` ツールで追う（サーバの設定によっては
ジョブにならず、同期で待ってから結果を返す）。

#### 台本→MP4 の一括生成もジョブで受け付ける

`create_yukkuri_video`（`POST /api/v1/agent/generate`）の `output:"mp4"` も、
`Prefer: respond-async` を付けるとジョブになる（サーバの設定によっては付けてもジョブにならず、
同期で待ってから 200 と `renderId` を返す）。
`output:"draft"` は常に同期。**MCP の `create_yukkuri_video` は自分でこれを付ける**ので、
MCP から使うときは何もしなくてよい。

受け付けでは、同期と同じ門（認証・スコープ・プラン・レート上限・`callbackUrl`・
クレジット）をその場で通す。残高不足・プラン不足・レート上限などは**数秒でその場に返り**、
そのときプロジェクトもジョブも作られない。通ったらクレジットを押さえ、プロジェクトと
台本を作ってから 202 を返す。本番の MP4 はこの時点で音声が無く長さが分からないので、**台本の文字数から多めに見積もって**押さえ、
音声を作ったあとに実際の長さで精算する（多く押さえた分は返す）。文字数は各行の `text` と `reading` の**長いほう**で数える
（音声は `reading` があればそちらを読むため。`text` のルビ `｜漢字《よみ》` は記号を数えず、親文字とよみの長いほう）。実際の長さの額が見積もりを超え、差額を残高から引けないときは
**焼かずに** `failed`（`error.code: "insufficient_credits"`）になり、押さえた分は全額返る。音声は作成済みなので、
クレジットを足してからそのプロジェクトを `render_mp4` で焼けばよい:

```json
{ "jobId": "...", "status": "queued", "pollUrl": "/api/v1/jobs/{jobId}", "projectId": "..." }
```

流れ（MCP）:

    create_yukkuri_video { output:"mp4" }   # 数秒で jobId と projectId が返る
    get_job { jobId }                       # succeeded まで待つ（音声合成とレンダー開始）
    get_render { projectId, renderId }      # renderId は get_job の result.renderId

- **202 には `renderId` が入らない。** `jobId` を `get_render` に渡さないこと。
- `succeeded` は**レンダーの開始まで済んだ**という意味で、動画はまだ焼いている途中。
  `result` は同期の 200 と同じ形（`projectId`・`renderId`・`status: "rendering"`）。
- `failed` なら `error.code` に理由が入り（`missing_audio`・`too_many_concurrent_renders`・
  `plan_required`・`insufficient_credits` など）、押さえたクレジットは**返金される**（反映まで少しかかることがある）。
  例外は、**レンダーを起動したあとにジョブだけが失敗した**場合（まれ）。このときレンダーは
  課金されたまま動いていて、返金されない。`get_render` に `projectId` と
  `renderId: "agent-{jobId}"` を渡せば結果が分かる（`not_found` なら起動しておらず、返金される）。
  MCP の `get_job` は本文でどちらかを案内する。
- 同時に走らせられるレンダーの上限に当たっていると、ジョブは15分ほど空きを待ち、
  それでも空かなければ `too_many_concurrent_renders` で失敗して返金される。
- `callbackUrl` を渡したときは、202 に `callbackSecret` と `callbackSignatureHeader` が入る。
  **ジョブの GET（`get_job`）には入らない**ので、この応答で控えること。
- ジョブの受け付けには `Authorization: Bearer` が要る。スコープは `render:mp4` に加えて
  `audio:generate` も要る。
- 同じ `idempotencyKey` で投げ直すと、ジョブが処理中・成功済みなら**同じ jobId** が返り、
  課金は起きない。失敗したジョブの鍵は空くので、理由を直してから同じ鍵でやり直せる。
  ただし上の例外（起動後にジョブだけが失敗した）では鍵は空かず、同じ鍵で投げ直すと
  起動済みのレンダーの `renderId` が 200 で返る（2本目は作られない）。
- `Prefer` を付けない REST の呼び出しは同期のまま。ただしサーバの設定によっては、
  同じ受け付けのあと一定時間だけ完了を待ち、終わらなければ `Preference-Applied` なしの
  202 を返す。**202 が返りうるものとして扱うこと。**

### YouTube へ投稿する（消費なし）

焼き上がったら `upload_to_youtube` で投稿できる。**ただし条件が4つある**:

1. **AI から投稿できるのは、クレジットを使って書き出した MP4 だけ**（`render_mp4` か `create_yukkuri_video` の
   mp4 で焼いたもの）。`renderId` のレンダーが完成・課金済み（`creditsCharged > 0`）・返金なしでなければ
   **403 `render_not_paid`**（`reason`: `not_charged` / `refunded` / `failed` / `not_completed`）、
   焼いている途中なら **409 `render_not_ready`**（`get_render` で `done` かつ `outputFile` を確かめてから呼び直す）。
   共有リンク（`create_preview_link`）だけで MP4 を書き出していないなら、先に `render_mp4`（動画1分につき1クレジット）を
   案内するか、利用者に画面（左のメニューの「動画」）から自分で投稿してもらう——**画面からの投稿はこの制限を受けない**
2. **鍵に `youtube:upload` スコープが要る。** 既定では付いていない（403 `insufficient_scope`）。利用者に、
   API キーの画面（`https://app.yukkurigen.com/settings/api-keys`）で「YouTube への投稿も許可する」を選んで
   鍵を発行し直してもらう。コネクタ（OAuth）でつないでいるなら、つなぎ直して許可画面で同じ項目を選んでもらう
3. プロジェクトに**チャンネルが紐づいている**こと（未分類は不可）
4. そのチャンネルで **YouTube 連携が済んでいる**こと（未連携なら 400 で理由が返る）

プランは問わない。**クレジットパックだけの無料プランでも、クレジットを使って書き出した MP4 なら投稿できる**
（有料プランか API 連携ライセンスの門は掛けない。決め手は「クレジットを消費して書き出したか」だけ）。

**回数の上限がある**（YouTube の動画アップロードの枠は YukkuriGen 全体で1日100回で、
利用者全員で分け合っている）: **1人1日10本まで**（直近24時間）、**同じ `renderId` は3回まで**
（どちらも失敗した分は数えない）、**失敗も含めた試行は1人1日20回まで**。超えると 429
（1日の上限・試行の上限なら `Retry-After` に待つ秒数）。同じ動画を何度も上げ直さないこと。

**公開範囲の既定は `private`**（本人しか見られない下書き）。利用者が公開を
望んでいると確認できたときだけ `public` にすること。投稿は非同期なので、
返った `jobId` を `get_youtube_upload` で追う。`completed` になれば
`videoId` が入る。

リンクは既定7日で切れる（最大30日）。渡す相手を間違えたら `revoke: true` を
付けて作り直す。**以前配ったURLは全部開けなくなる。**

なお、プレビューはブラウザで組み立てて再生している。書き出す側とは
プログラムの配布経路が違うので、**細部は完全に同一ではない**。最終的な
見た目は書き出した MP4 で確かめること。

### 上げた動画の実績を見て、次の台本を良くする（消費なし）

YouTube 連携済みのチャンネルなら、上げた動画の実績を読める（クレジットも投稿の枠も使わない）:

- `get_channel_video_analytics`（`channelId`・`days` 既定90・`limit` 既定50）: 動画ごとの再生・総視聴時間・
  平均視聴時間・平均視聴率・高評価・コメント・共有・登録者増を、再生の多い順に。YukkuriGen で作って
  上げた動画には `projectId` が付く——`get_project` で台本を読み返し、伸びた回と伸びなかった回の
  テーマ・題名・長さ・冒頭を比べる
- `get_video_retention`（`channelId`・`videoId`）: 動画を100等分した各点の維持率と、いちばん大きく
  減った3か所の秒数。**その秒数あたりの台本の行**を見て、冒頭のつかみや話の順番を直す

**表示回数とクリック率（サムネイルと題名の強さ）は YouTube の API に無いので取れない**（YouTube Studio で見る）。
当日分は入らず、公開から数日は数字が少ない。再生の少ない動画は維持率が返らないことがある。
直すときは一度に1つだけ変える（何が効いたか分かるように）。
チャンネルが YouTube と連携されていなければ 400 `youtube_not_connected`——利用者に、画面のチャンネル設定から
YouTube を連携してもらう。

### 応答が返らないときのために（重要）

`create_yukkuri_video` の `mp4` を**同期で**呼ぶと、**音声合成とレンダー開始を
待ってから返る**。10行の台本で45秒を超えることがあり、クライアントによっては
そこで切れる。切れても**サーバ側では課金もレンダーも進んでいる**ので、
何もせず投げ直すと二重に払うことになる。

- **ジョブで受け付けてもらう。** MCP の `create_yukkuri_video` は自分でそうするので、
  数秒で `jobId` が返る（サーバの設定によってはジョブにならず、同期で待ってから `renderId` を返す）。
  REST を直接叩くなら `Prefer: respond-async` を付ける（上の「非同期ジョブ」）
- **`idempotencyKey` を必ず付ける。** 同じ鍵で投げ直せば、最初の結果（ジョブなら同じ
  `jobId`）が返り課金は起きない。同期の処理中なら「進行中」と返るので少し待って
  同じ鍵で再試行する
- 段ごとに刻みたいなら **`output:"draft"` で刻む**:

      create_yukkuri_video { output:"draft" }   # 消費なし・数秒で返る
      generate_audio                            # 消費なし
      render_mp4                                # ここで初めて課金（動画1分につき1クレジット）
      get_render                                # 完了までポーリング

  1回あたりが短くなるので切れない。`draft` は `projects:write` だけで通る
  （`render:mp4` は要らない）。

### 縦型で作りたいとき

`create_yukkuri_video` の `platform` に `shorts`（または `tiktok`）を渡す。
既定は `youtube`（横型）。**向きは `platform` だけで決まる**（`outputFormat` は `platform` を省いたときの手がかりにしか使わない）。

`platform` は**プロジェクトに残る**。draft で作ったあとや直したあとの `render_mp4` で `platform` を省くと、
作ったときの向きで焼かれる（`get_project` の `platform` で確かめられる）。向きを変えて焼き直したいときだけ
`render_mp4` に `platform` を渡す。共有リンクも、縦型のプロジェクトは縦型で描かれる。

縦型（1080×1920）は縦の画面いっぱいに組まれる: **上にタイトル（画面の上 約15%）→ 素材（16〜46% あたり）→
キャラ2人を左右に大きく（46% より下。4人そろうと横に4人並ぶ）→ 字幕（74〜90%）**。字幕は横長より大きい。
縦型では素材を `materialBox` で置くなら **y は 16 以上・y+height は 48 以下**が目安（タイトルと顔を避ける）。
縦型の配置を直すときは `update_template_layout` に `"aspect":"9:16"`（今の値は `list_templates` の `portraitLayout`）。

### 見た目の既定はチャンネルに置ける

文字サイズ・素材フィルター・エンディングカードは、チャンネルの設定画面
（`/channels/{id}` の「設定」タブ）で既定を決められる。1本だけ変えたいときは
`render_mp4` の引数が優先される（**リクエスト > プロジェクト > チャンネル**）。
字幕の文字サイズだけは、テンプレートに決めた大きさ（`update_template_style` の `subtitle.fontSize`）が
チャンネルの既定より優先される（**リクエスト > プロジェクト > テンプレート > チャンネル**）。

## 手順

1. **キャラを選ぶ**: `list_characters` で `id` と音声エンジンを確認。
   **1本の動画に出せるのは同じ一座のキャラだけ**（立ち絵の系統が違うと画面に出せない）:
   - ゆっくり（東方）: `reimu`（霊夢）, `marisa`（魔理沙）
   - 東北勢: `zundamon`（ずんだもん）, `metan`（四国めたん）, `tsumugi`（春日部つむぎ）, `anko`（あんこもん）

   混ぜると `400 cast_mismatch` を返す（課金なし）。応答の `casts` に一座の一覧が入っている。

   **利用者の自作キャラ**（「自分のキャラ」の画面 `https://app.yukkurigen.com/assets` で PSD から取り込んだ立ち絵など）も `list_characters` に
   `custom: true` で出る。その `id`（20文字の英数字）をそのまま `speaker` に書けば話者になる。
   一座は東北勢と同じ（ずんだもん等とは一緒に出せる。ゆっくりとは混ぜられない）。`emotions` / `poses` は
   そのキャラに設定済みの表情・ポーズで、無い表情を書いても近い表情か通常の顔で描かれる。
   利用者に「この画像を自分のキャラにして」と頼まれたら、**あなたが `create_character` で作れる**
   （PNG・WebP の画像1枚から。下の「自分のキャラを作る」。PSD の取り込みだけは画面）。

   **テンプレート（見た目）は `list_templates` から選ぶ。** 返った `id` を `templateId` に渡す。
   省略すれば台本の話者から自動で選ぶので、指定は必須ではない。利用者が「いつもの見た目で」と
   言ったら `source: "mine"` から選ぶこと。
   - **話者のいる動画に使える組み込みは `yukkuri-talk`（reimu・marisa）と `tohoku-ahiru-talk`
     （東北勢・自作キャラ）だけ。** 組み込みは5種あるが、`line-scroll`（コメント型）・`youtube-bgm`（BGM 用）・
     `story-talk`（いらすとや）は立ち絵の一座が違う
   - `source: "mine"` は、`basedOn` が台本の一座と同じもの（ゆっくりなら `yukkuri-talk`、東北勢なら
     `tohoku-ahiru-talk`）を選ぶ。`basedOn` の無い（白紙から作った）テンプレートは立ち絵の検査ができない
   - **見た目を変える画面は無い（テンプレートエディタは 2026-10-14 に廃止）。** 位置（キャラ・字幕・素材の枠）も、
     字幕の色・大きさなどの見た目も、利用者に頼まれたらあなたが直す（下）。直すこと自体にクレジットはかからない
   - 作ったあとにテンプレートを替えるのは `set_project_template`（台本・音声はそのまま）

   存在しない id や他人のテンプレートを渡すと `400 template_not_found` を返す
   （課金なし）。**黙って別の見た目で焼くことはしない**——2026-09-09 まではそう
   なっていて、自作テンプレートを指定しても `line-scroll`（スポーツ反応集）で
   焼かれていた。

   チャンネル（テーマ・想定視聴者・既定の指示）を使うなら `list_channels` で
   `id` を取り、`channelId` に渡す。**見た目の優先順は「指定した `templateId`（組み込みでも自作でも）＞
   チャンネルのテンプレート＞台本の話者から自動」。** `list_channels` の `templateId` が、そのチャンネルで
   指定なしのときに当たるテンプレート。

   **素材の位置と大きさは、行ごとにあなたが決めてよい**（`materialBox`。下の「背景を決める」）。

   **1本の動画の中で行ごとにキャラの立ち位置を変えたいとき**（1人で真ん中・左右の入れ替え・奥に小さく）は、テンプレートではなく
   行の `stage` に `{"id","x","y","scale","facing"}` を書く（下の「場面の演出」）。

   **テンプレートの位置を変えたいとき（「キャラを上に」「字幕を上に」「素材の既定の場所を下に」）も、あなたが直せる**:
   1. `list_templates` で今の `layout`（横長）/ `portraitLayout`（縦型）を読む（キャラの枠・字幕・素材の枠。画面に対する %、x/y は左上）
   2. 組み込み（`source: "system"`）は直せないので `copy_template` で複製する。`yukkuri-talk` と `tohoku-ahiru-talk` の複製は
      **元の見た目のまま**描かれ、位置だけが変わる（ほかの3種の複製は見た目も変わる）
   3. `update_template_layout` で直す（渡した項目だけ変わる。縦型は `"aspect":"9:16"`）。**値は差分ではなく新しい値**
      （今の値に足した結果を渡す）。例: 素材の既定の枠を下げる（今 y:5 なら）`{"material":{"y":15}}`、
      キャラを上げる（今 y:4 なら）`{"characters":[{"slot":"left1","y":-2}]}`。
      東北の立ち絵の枠は画面の下にはみ出している（`y+height>100`）——腰から下を切るための仕様なので、画面に収めようとしないこと（収めるとキャラが小さくなる）。
      **キャラを常に定位置に固定したいとき**は `{"keepCharactersInPlace":true}` を渡す（全画面素材と card 行だけの動画向け）。今の値は `layout.keepCharactersInPlace`。
   4. 作った動画に当てるなら `set_project_template`、これから作るなら `create_yukkuri_video` の `templateId`
   5. `create_preview_link` の共有リンクで見て、良ければ焼く

   キャラの立ち絵は枠の下端にそろい、枠の高さで大きさが決まる。素材は枠に収まる大きさで枠の上端に出る。

   **見た目を変えたいとき（「字幕を黄色に」「字幕を大きく」「話者名を出して」「BGM を小さく」「背景をこの画像に」）も、あなたが直す**:
   1. `list_templates` の `style` で今の値を読む（`GET /api/v1/agent/templates/{templateId}/style` でも同じ）。
      `style.unsupported` は**そのテンプレートでは描かれない項目**で、渡すと `400 unsupported_field`（直さない）
   2. 組み込みは `copy_template` で複製してから、`update_template_style` で直す（渡した項目だけ変わる。値は新しい値）
   3. `set_project_template` で動画に当て、`create_preview_link` で見て、良ければ焼く

   **字幕の色・フチ・箱は `subtitle.look`（行の字幕の描き方）で効き方が変わる。** AI で作った動画の既定は `bold`
   （白の太字に話者の色の太フチ・箱なし）で、`bold` では `color`・`fontWeight`・`outlineColor`・`outlineWidth`・
   `backgroundColor`・`lineHeight` は効かない（効くのは `fontSize`・`textAlign`・`letterSpacing`・`speakerName`）。
   文字の色を変えるなら `look` も一緒に変える:

   | `look` | 描き方 | 効く色 |
   |---|---|---|
   | `bold` | 白の太字・話者の色の太フチ・箱なし | なし（白固定） |
   | `box` | 箱に文字（フチなし） | `color`・`backgroundColor` |
   | `box-outline` | 箱に文字・フチ | `color`・`outlineColor`/`outlineWidth`・`backgroundColor` |
   | `box-speaker-outline` | 箱に文字・話者の色のフチ | `color`・`backgroundColor`（フチの太さは `outlineWidth`） |
   | `frame` | 白い箱・黒い文字・話者の色の枠 | なし |

   例: 黄色い字幕に黒いフチ `{"subtitle":{"look":"box-outline","color":"#ffd400","outlineColor":"#000000","outlineWidth":6}}`、
   字幕を大きく `{"subtitle":{"fontSize":72}}`（既定は yukkuri-talk 60・tohoku-ahiru-talk 56。長い行は枠に収まるよう自動で縮む）、
   BGM を小さく `{"bgm":{"volume":0.15}}`（0.3 が標準）、話している間だけ BGM を下げる `{"bgm":{"ducking":true}}`。
   応答の `notes` に、今の組み合わせで効かない項目が書かれる。

   直せる項目（`update_template_style`）:
   - `subtitle`: `look`・`fontSize`（px、横長 1920 幅で。24〜160。`null` で既定）・`color`・`fontWeight`（400/700/900）・
     `outlineColor`・`outlineWidth`・`backgroundColor`（`null` で話者の色の半透明）・`backgroundPadding`・`backgroundBorderRadius`・
     `textAlign`・`letterSpacing`・`lineHeight`・`speakerName`（`none` / `inline` / `above` / `badge`）。
     色は `#rrggbb`・`#rrggbbaa`・`rgb()`・`rgba()`
   - `characters[]`: `slot` ごとに `voiceOnly`（その**枠**を声だけにする。テンプレートを使う全部の動画に効き、枠は空かない）・`flip`（左右反転。自作キャラにだけ効く。組み込みのキャラは自動で向かい合わせる）。
     **1本の動画で特定のキャラを声だけにするなら、こちらではなく動画の `voiceOnly`**（「声だけの話者」。枠を使わず、空いた枠はほかの話者が使う）
   - `camera`: `zoomScale`（行の `cameraMode:"dynamic"` で寄る倍率、行の `camera` で `zoom` を省いたときの倍率。1〜2、既定 1.4）・`backgroundBlurOnZoom`
   - `bgm.volume`（0〜1）・`bgm.ducking`（`true` で話している間だけ BGM を下げる。既定 `false`。「BGM と効果音」）・
     `background`（`imageUrl` / `videoUrl`。プロジェクト・行に背景が無いときの既定。`null` で外す）・`name`

   `yukkuri-talk` を元にしたものは `camera`・`flip` を描かない（寄りは固定）。`yukkuri-talk` / `tohoku-ahiru-talk` を元にしたものは
   字幕の影・アニメーション・単色の背景を描かない。それ以外（ほかの組み込みの複製）は `look` を使わず書式をそのまま描く。
   どれも `style.unsupported` に出ている。字幕の大きさは `render_mp4` の `fontSize` とプロジェクトの `fontSize` が
   テンプレートより優先される（**リクエスト > プロジェクト > テンプレート > チャンネル**）。
   書体（フォント）は選べない（描く側にフォントを読み込んでいないため）。

   **返るのは MP4。** こちらで音声を合成してから本番レンダーを開始し、
   `renderId` を返す（動画1分につき1クレジット・有料プランかパックで買ったクレジット）。焼く前に見せたいなら `output:"draft"` で作り、
   `create_preview_link` の共有リンクを渡す（0クレジット・無料プランでも可）。
   ジョブで受け付けたとき（MCP など）は先に `jobId` が返り、`get_job` の
   `result.renderId` で受け取る（上の「非同期ジョブ」）。
   どちらも最後は `get_render` でポーリングする。
2. **台本を書く**: 話者(`speaker`=キャラid)とセリフ(`text`)の配列を作る。掛け合い形式が「ゆっくり」らしい。
   - **読み違えそうな漢字にはルビを書く**: `｜今日《こんにち》は`（字幕は「今日は」、声は「こんにちは」）。
     人名・作品名など何度も出る語は `set_pronunciation` で辞書に入れる。台本を作ったら、音声の前に
     `check_readings` で実際の読みを確かめる（下の「読みを確かめて直す」）。行全体を読み替える `reading`（発音かな。
     1行2000文字まで）は、ルビで足りないときだけ（その行は辞書もルビも効かなくなる）。
   - 冒頭の挨拶 → 本題（結論→理由→具体例）→ まとめ、の構成が定番。
   - **独自の意見・体験を1つ以上入れる**と、量産テンプレに埋もれない動画になる。
   - **YouTube で伸びている解説動画の形**（2026-09 に人気動画と見比べた）: 5〜10分（60〜150行）。
     最初の3行で「え、そうなの？」と思わせる引き（意外な結論・問いかけ）→ 章ごとに1つの話題
     （1章 10〜25行）→ 最後に要点のおさらい → 「チャンネル登録してね」の一言で締める。
     1行は60文字以内が目安（テンプレートの字幕ボックスで通常サイズに収まる上限）。60〜87文字だとフォントが自動縮小、88文字以上だと下限サイズでも収まらない可能性がある。`get_project` の応答に `subtitleWarnings` が含まれる場合は、その行を分割か短縮すること。聞き手のツッコミ・驚きを数行ごとに挟む。
   - **章の見出し・強調テロップ・写真・図解は、あなたが行ごとに書く**（下の「図解・写真・章・強調はあなたが決める」）。
     書かなければ出ない——YouTube で伸びている解説動画はどれも入れているので、必ず書くこと。
3. **背景を決める**: `backgroundImageUrl`（https のURL）を渡す。
   省略すると既定の背景（室内のイラスト）になる。話題に合う背景を渡すと見栄えが上がる。
   場面転換を付けたいときは、その行の `backgroundImageUrl` に別のURLを入れる（以降の行へ引き継がれる）。
   利用者が用意した写真・スクショを行ごとに出したいときは、その行の `materialImageUrl`（https）に入れる（その行だけ。`update_lines` でも付け外しできる）。
   **素材の位置と大きさは、行ごとにあなたが決める**（`materialBox`。画面に対する %、x/y は左上）。
   省略するとテンプレートの素材の枠（組み込みの横長は上寄りの中央 `{x:23,y:5,width:54,height:60}`）。
   画像は枠に収まる大きさで、枠の上端・左右中央に出る（縦横比は保つ）。photo・card の画像にも効く。
   ```json
   {"title":"写真の置き方","script":[
    {"speaker":"reimu","text":"この写真を見て。","materialImageUrl":"https://example.com/a.jpg",
     "materialBox":{"x":35,"y":8,"width":30,"height":35}}]}
   ```
   - 横長で**顔にかからないのは x が 20〜83% の間**（左右のキャラの顔はその外側）。字幕は下の約15%。
     目安: 既定の枠 `{x:23,y:5,width:54,height:60}` / 中央に小さく `{x:35,y:8,width:30,height:35}` /
     大きく見せる `{x:20,y:3,width:60,height:68}`
   - 縦型: y は 16 以上・y+height は 48 以下（上のタイトルと下の顔を避ける）。例 `{x:5,y:17,width:90,height:28}`
   - 値の範囲: x・y は -50〜100、width・height は 0 より大きく 150 まで（外れると 400）
   - **画面全体を覆う素材**は `{x:0,y:0,width:100,height:100}`（16:9 の画像なら画面いっぱい）。
     この行ではキャラは**寄らず元の位置のまま**、素材の後ろに隠れる（素材は既定でキャラより前）。
     全画面でない素材の行は、キャラが少し外側へ寄って画面を空ける（テンプレートに `keepCharactersInPlace:true` を設定すると常に寄せない）
   - 同じ画像を続けて出す行は同じ値にする（値が変わると、そこで画像が出し直される）。
     card は `span` 行ぶん、photo は次の行にも続けて出るが、その続く行には元の行の `materialBox` が自動で写る
   - あとから動かすなら `update_lines` に `{"index":3,"materialBox":{...}}`（`null` でテンプレートの枠に戻す）
   BGM は指定しなくても既定の曲が流れる。行ごとに曲を変える・止める・自分の曲を使うなら行の `bgm`（「BGM と効果音」）。
   `title` の出方: 横長は左上に出す。ただし**章の見出し（`chapter`）が1つでもあると、左上は章の見出しになり、
   タイトルは最初の章より前の行だけに出る**（1行目に章を書くとタイトルは出ない）。縦型は画面の上にタイトルを
   ずっと出し、章の見出しは出さない。`titleTelop: false` は横長・縦型どちらのタイトルも消す。
   **図解・写真・章・強調はあなたが決める**: サーバは台本から勝手に作らない（台本や演出は
   サーバで生成しない。LLM は呼ばない）。台本を書いたあなたが、行ごとに次の項目を書く。どれも任意で、書いた行にだけ出る。
   - `chapter`: その行から始まる**章の見出し**（14文字以内が目安。30文字まで受け付け、16文字を超えた分は黙って切る——エラーにならない）。横長の画面の左上に次の章まで出る（縦型では出ない）。
     話題の区切りごと、目安は8行に1つ。タイトルを冒頭に見せたいなら、最初の章は2〜3行目に書く。
   - `card`: **図解カード** `{ "title": "…", "items": ["…","…"], "style": "list" }`。要点のまとめ・
     手順・理由・比較・数字など、図にすると分かりやすい所に（目安: 7行に1枚、最大6枚）。
     `style` は `list`（箇条書き）/ `steps`（番号付きの手順）/ `point`（要点・数字を1つ大きく、項目1つ）。
     `span` 行ぶん出る（既定3、次のカードの手前まで）。項目は台本にある事実だけ、短い体言止めで。
   - `photo`: その行に添える**写真の英語の検索語**（1〜3個）。話に出てくる具体的でありふれた物・場所を、
     具体的なものから順に、最後は1語で（例 `["coffee cup", "coffee"]`、`["alarm clock", "clock"]`）。
     題名に全部の語が入った写真だけ使うので、抽象語（taste, morning 等）は写真にならない。
     目安: 4行に1枚。Openverse の CC0・パブリックドメインの写真を探して入れる（見つからなければ出ない）。
   - `emphasis`: その行で画面の中央に大きく出す**強調テロップ**（台本にある要の言葉、10文字以内が目安。30文字まで受け付け、12文字を超えた分は黙って切る——エラーにならない）。
     `"夜のスマホ"` または `{ "text": "危ない", "color": "red" }`（yellow=既定 / red=危険・注意 /
     blue=冷静・事実 / green=良い・安心）。目安: 6行に1つ。**素材（materialImageUrl・photo・card の画像）や
     図解カードが出ている行では出ない**（カードが続く行も含む）。
     **出す場所もその行だけ決められる**: `{ "text": "危ない", "position": "top" }` の `position` は
     `top`（画面の上の方）/ `center`（画面の真ん中）/ `bottom`（字幕のすぐ上）。省くと従来の高さ（横長は上から26%、
     縦型は24%。顔と顔の間）。細かく決めるなら `x`・`y`（テロップの中心。画面の幅・高さの %、0〜100）。
     `y` は `position` より勝つ。`x` を端に寄せるほど、はみ出さないよう折り返す幅が狭くなる（20〜80 の間が無難）。
     例: 右上なら `{ "text": "3倍", "x": 72, "y": 18 }`。次の行へは引き継がない。
     縦型では `center`・`bottom` はキャラの顔・体に重なる（縦型は従来の高さか `top` が無難）。

   カードと写真は、**共有リンクを作るとき（`create_preview_link`）か、焼くとき（プレビュー含む）**に画像にして
   行の**素材**（`material_image_url`）に入れる。それまでは共有リンクにも出ない。画像にしたあとは `get_project` で
   見えて、`update_lines` や画面の素材の列で差し替え・削除できる。画像にできなかったカードは、画面の上寄りに重ねて描く。
   `chapter`・`emphasis`・`card`・`photo`・`effect`・`bgm` は、作ったあとでも `update_lines` で足せる・外せる（`null`）。
   素材がすでにある行に card・photo を足すと、前の素材は外れて新しい画像に替わる。

   ```json
   {"title":"コンビニコーヒーが安い理由","script":[
     {"speaker":"zundamon","text":"安くできる理由は三つあるのだ。","chapter":"安さの3つの理由",
      "card":{"title":"安くできる3つの理由","items":["豆をまとめて大量仕入れ","ボタンひとつで自動抽出","お客さんのセルフ式"],"style":"steps","span":4}},
     {"speaker":"metan","text":"香りはどうなの？","photo":["coffee beans","coffee"]},
     {"speaker":"zundamon","text":"人の手間を減らすのが一番の近道なのだ。","emphasis":"手間を減らす"}
   ]}
   ```
   **クレジット**: 完成した `get_render` の結果に、YouTube の概要欄に貼るクレジット（`credits`）が付く。
   VOICEVOX の音声は「VOICEVOX:ずんだもん」のような表記が利用規約の条件なので、**必ず概要欄に入れること**
   （`upload_to_youtube` は自動で足す）。
4. **カメラ（任意）**: 寄りは**入れなくてよい**。入れるなら、盛り上がり・感情のピーク・オチの行に
   `cameraMode: "dynamic"` を付けると、**背景ごとカメラがその話者に寄る**。立ち絵を消したい行（場面の要約など）は
   `"summary"`。`null`（または省略）は寄らない。
   **使うなら2〜4回に留めること。全行に付けると寄りっぱなしになり、寄りが効かなくなる。**
   立ち絵が画面に出ている行にだけ効く。
   **寄りを一切入れたくないときは、どれか1行に `"cameraMode": null` と書く**（下の自動演出が止まる）。
   作った後なら `update_lines` で寄っている行を `null` にすれば寄らなくなり、後から自動で付け直されることはない。

   **行ごとのカメラ `camera`（任意）: 寄る先を画面の真ん中に持ってきて寄る。** `cameraMode:"dynamic"` は話者の
   顔を中心に拡大するだけなので、左端に立つキャラは左端のまま大きくなり、もう1人が画面の端で半分切れる。
   「このキャラを真ん中に大きく」「ここで画面を揺らして」と言われたら、その行に `camera` を書く。
   **キャラは動かさず、場面（背景・キャラ・素材）ごとカメラが動く**（キャラ同士の距離はそのまま。離れて立つ
   もう1人は画面の外へ出る）。
   ```json
   { "edits": [ { "index": 5, "camera": { "target": "zundamon", "zoom": 2 } },
                { "index": 6, "camera": { "shake": "strong" } } ] }
   ```
   （`update_lines` の例。左に立つ自作キャラ（部長など）を真ん中に大きく映すなら、`target` にそのキャラの id を書く:
   `"camera": {"target": "<部長の id>", "zoom": 2}`。作るときは `script` の行に同じ `camera` を書く）
   - `zoom`: 寄りの大きさ（1〜2.5。1 は寄らない）。省くと、`target` を書いたときはテンプレートの寄りの大きさ
     （`update_template_style` の `zoomScale`。既定 1.4、`yukkuri-talk` は 1.5）、書かなければ 1
   - `target`: 寄る先。`"speaker"`（その行の話者。既定）/ キャラの id（**台本で話すキャラだけ**。それ以外は
     `400 validation_error`、`invalidCameraTarget` に id）/ `{"x":75,"y":25}`（画面の %、0〜100 の点）。
     キャラへは**顔の中心**を狙い（組み込みキャラは絵から測った位置、自作キャラは `list_characters` の `face`）、
     いま立っている所（`stage` の立ち位置・4人の並び）へ寄る。自作キャラで顔が真ん中から外れたら `update_character` の `face` で直す
   - `center`: 寄る先を真ん中へ持ってくるか（既定 `true`）。`false` は `cameraMode:"dynamic"` と同じ
     「その点を中心に拡大」（寄る先は画面の同じ場所に残る）
   - `move`: 前の行のカメラからの移り方。`cut`（行の頭で一瞬で）/ `ease`（0.5秒でなめらかに。既定）/
     `slow`（寄る先へは0.5秒で向き、寄りの大きさは行いっぱいをかけてじわじわ変える）
   - `shake`: 行の頭で画面を揺らす。`true`・`"light"`（0.5秒の軽い揺れ）/ `"strong"`（0.8秒の大きな揺れ）。
     揺らす行は背景の外が映らないよう、わずかに寄る（1.6%・4.2%）
   - `materials`: その行の素材（`materialImageUrl`・`photo`・`card` の画像）を `"zoom"`＝場面と一緒に寄せる /
     `"fixed"`＝画面に止めたまま（`materialBox` の位置・大きさのまま）。省くとテンプレートの従来どおり
     （`tohoku-ahiru-talk`・`yukkuri-talk` とその複製は `zoom`、白紙から作った自作テンプレートは `fixed`）。
     寄った行で素材を読ませたいなら `"fixed"`。強調テロップ・字幕・章の見出しは常に止まったまま
   - **その行だけに効く**（次の行へ引き継がない）。次の行に `camera` が無ければ、その行の頭で引きの画に戻る（cut）。
     なめらかに引くなら次の行に `{"zoom":1,"move":"ease"}`。`cameraMode` と両方書いたら `camera` が勝つ。
     `update_lines` で `null` にすると外れる。範囲の外・知らない項目・空の `{}` は `400 validation_error`
   - **背景の外は映さない**ので、画面の端に立つキャラは真ん中までは来ない（寄るほど真ん中に近づく）。
     左右に離れて立つ2人（`tohoku-ahiru-talk` の既定の枠）で片方だけを映すなら **`zoom` 1.6 以上**
     （1.5 以下だと、もう1人が画面の端に少し残る）。近くに並ぶ3〜4人なら、となりのキャラが少し映るのは自然
   - **キャラを真ん中に映すために `stage` の `x:50` で立ち位置を動かさない。** 立ち位置はそのままで、
     `camera` の `target` で寄る
   - **使いどころ**: リアクション・ツッコミ・オチ・驚き・決めゼリフの行だけ（目安: 10行に1〜2行。全行に書くと
     寄りが効かなくなる）。例: 怒る部長を真ん中に大きく `{"target":"<部長の id>","zoom":2}` / 驚きで画面を揺らす
     `{"shake":"strong"}` / 語りの山場でじわじわ寄る `{"zoom":1.6,"move":"slow"}` / 画面の右上の物を見せる
     `{"target":{"x":75,"y":25},"zoom":2}`
   - `tohoku-ahiru-talk`・`yukkuri-talk`（とその複製）・白紙から作った自作テンプレートで効く（`story-talk` では効かない）。
     `camera` を1行でも書けば、作るときの自動の寄り（`cameraMode` の自動）は付かない

   **場面の演出（任意）: 舞台の出入り・背景の動き・場面転換・アイキャッチ。** 利用者が「ゲストを途中から出して」
   「背景をゆっくり動かして」「章の頭にタイトルを大きく」と言ったら、テンプレートではなく**台本の行に書く**。
   どれも書かなければ以前と同じ見た目のまま（既定は「全員が初めて話した行で登場して最後まで残る・背景は止まる・
   背景の切り替えは 0.6 秒で溶かす・アイキャッチ無し」）。作ったあとも `update_lines` / `insert_lines` /
   `split_line` で足せる・外せる（`null`）。どれを変えても音声は消えない。
   - `stage`: その行から**舞台に立つキャラの id の並び**。**次に `stage` を書いた行まで引き継ぐ。**
     並びから外したキャラはその行で退場し（外側へすべって消える）、加えたキャラは登場する（外側からすべって入る）。
     ゲストを途中から呼ぶ・役目を終えたキャラを下げる・相方を下げて1人で話す、に使う。
     書けるのは**台本で話すキャラだけ**（それ以外は `400 validation_error`、`invalidStage` に id）。
     **台詞のある行の話者は、並びに無くてもその行で登場し、以後も残る**（下げたいなら後の行の `stage` から外す）。
     ただし行の `offstage: true`（声だけの行）の話者は登場しない。動画の `voiceOnly` のキャラは書けない
     （`400 validation_error`、`voiceOnlyOnStage` に id。下の「声だけの話者」）。`[]` は「その行の話者だけ」。
     **立ち位置もあなたが決めてよい**: 並びの要素を `{"id":"zundamon","x":50,"y":0,"scale":1.1,"facing":"left"}` のような
     オブジェクトにすると、そのキャラはテンプレートの枠ではなく書いた所に立つ（文字列と混ぜてよい。`id` 以外は省略可）。
     - `x`: キャラの**横の中心**（画面の幅に対する %、0〜100。50 が真ん中）。枠の既定の中心は `tohoku-ahiru-talk` の横長が
       左から およそ 16・43・57・82、縦型が 25・40・60・75、`yukkuri-talk` の横長が 17・83
     - `y`: 枠の位置から**下へずらす量**（画面の高さに対する %、-50〜50。マイナスで上へ。既定 0）
     - `scale`: 枠に対する**大きさ**（0.5〜2。既定 1）。`tohoku-ahiru-talk` と自作テンプレートは**頭の高さを保ったまま**
       大きく・小さくなり、`yukkuri-talk` は下端を保つ。奥に小さく立たせるなら `y` も足して下げる（例 `scale:0.75, y:10`）
     - `facing`: **顔の向き**（`"left"` / `"right"`）。省くと、`x` を書いたときは画面の中央の方を向く（`x` が 45 未満なら右、
       55 を超えたら左、その間は枠の向き）。`x` も無ければ枠の向き。自作キャラは元の絵が左向きとして扱うので、
       逆を向いたら `facing` を反対に書く
     書かなかった項目は枠のまま。範囲の外・知らない項目は `400 validation_error`。
     **位置も `stage` と一緒に引き継ぐ**: 次に `stage` を書いた行では、文字列で書いたキャラ・並びに無いのに台詞で登場した
     キャラは枠の位置に戻る（位置を保つなら同じオブジェクトをもう一度書く）。舞台に居るまま位置・大きさが変わると
     約0.3秒でなめらかに動く（向きはその行の頭で切り替わる）。位置を書いたキャラの登場・退場は画面の近い方の外からすべる。
     位置を書いたキャラは**4人の並びにも素材の寄せにも従わず、書いた所に立ち続ける**（素材を出す行では、素材の枠と顔が
     重ならない `x` を選ぶ。横長の素材の既定の枠は画面の横 23〜77%）。`cameraMode:"dynamic"` はいま立っている所へ寄る。
     **1人で話すときに真ん中へ寄せるのは自動ではない**（`x:50` と書く）。ただし**カメラで真ん中に大きく
     映したいだけなら `x:50` で立ち位置を動かさず、行の `camera` の `target` を使う**（上の「行ごとのカメラ」）。
     例: 1人で真ん中 `[{"id":"zundamon","x":50}]` / 左右を入れ替える
     `[{"id":"zundamon","x":82,"facing":"left"},{"id":"metan","x":18,"facing":"right"}]` /
     奥に小さく `["zundamon",{"id":"anko","x":70,"scale":0.75,"y":10}]`
   - `backgroundMotion`: **背景のゆっくりした動き**。`none`（既定・止める）/ `zoomIn`（ゆっくり寄る）/
     `zoomOut`（ゆっくり引く）/ `panLeft`・`panRight`（カメラが左・右へ流れる）。**書いた行から、次に書いた行まで
     引き継ぐ**ので、動画全体に掛けるなら1行目に書く（止めるなら `none` を書く）。その背景が出ている間いっぱいを
     かけて最大8%動く（8秒より短い場面では小さく）。背景を差し替えた行・動きを替えた行から頭から動く。
     画像の背景だけに効く（動画の背景は動かない）。`cameraMode:"dynamic"` の寄りとは別物で、重ねて使える。
   - `transition`: その行での**場面転換の見せ方**（その行だけ）。`dissolve`（既定・前の背景から 0.6 秒で溶かす）/
     `fadeBlack`（暗転。行の頭の前後 0.4 秒ずつで黒に落ちて戻る。字幕・キャラも含む画面全体）/ `cut`（一瞬で切り替え）。
     `dissolve`・`cut` は、その行で `backgroundImageUrl` か `backgroundMotion` が変わるときだけ効く。
     **`fadeBlack` は背景を変えない行でも暗転する**（時間が飛ぶ所・話題をはっきり区切る所に）。
   - `eyecatch`: その行の前に挟む**全画面の章タイトル**。`"今夜からできる対策"` か
     `{ "title": "後半", "subtitle": "対策編" }`（見出し30文字・小見出し40文字まで。見出しは12文字以内が読みやすい）。
     背景を暗くぼかしてキャラを消し、大きな見出しを 1.8 秒出す（横長・縦型とも）。**その行の音声の前に 1.8 秒の
     無音の間を足してそこに出す**ので台詞とは重ならない——**そのぶん動画が 1.8 秒ずつ長くなり、本番 MP4 の
     クレジット（動画1分につき1）にも入る**（`create_yukkuri_video` の受け付けの見積もりにも入る）。
     同じ行で背景・`backgroundMotion`・`stage` を変えると、アイキャッチの裏で入れ替わる。
     目安: 大きな話題の区切りに2〜4回。`chapter` に同じ見出しを書くと、横長の左上の章の見出しともそろう。

   ```json
   {"title":"お金の基本","script":[
     {"speaker":"zundamon","text":"今日はお金の話をするのだ。","stage":["zundamon","anko"],
      "backgroundImageUrl":"https://example.com/room.jpg","backgroundMotion":"zoomIn",
      "eyecatch":{"title":"お金の基本","subtitle":"まずはここから"},"chapter":"お金の基本"},
     {"speaker":"anko","text":"今日はゲストがいるのだ。"},
     {"speaker":"metan","text":"四国めたんよ。よろしくね。","stage":["zundamon","anko","metan"]},
     {"speaker":"zundamon","text":"また来てほしいのだ。","stage":["zundamon","anko"]},
     {"speaker":"anko","text":"外に出てみるのだ。","backgroundImageUrl":"https://example.com/forest.jpg",
      "backgroundMotion":"panRight","transition":"fadeBlack"},
     {"speaker":"zundamon","text":"次の章なのだ。","eyecatch":"今夜からできる対策","chapter":"今夜からできる対策"}
   ]}
   ```
   **何人まで出せるか**: 動画全体の話者で数えて **`tohoku-ahiru-talk`（とその複製）は4人まで**、`yukkuri-talk` は2人、
   白紙から作った自作テンプレートは置いた枠の数。超えた話者は声と字幕だけ。**声だけの話者（`voiceOnly` のキャラと、
   `offstage` の行でしか話さずどの `stage` にも無いキャラ）は数えない**——ナレーター＋画面の4人、ができる。
   `tohoku-ahiru-talk` は**その時点で舞台に居る人数で並ぶ**: 2人は左右に分かれて向き合う、3人は2人と1人、
   **4人そろうと左2人・右2人**（少し小さくなる。素材・図解カードが出ている間は素材をよけて左右の端に2人ずつ寄る）。
   `stage` で人数が変わると約0.3秒で並び直す（4人目の枠は `right2`。`list_templates` の `layout` に出る）。
   2026-10-15 より前に複製した東北のテンプレートは枠が3つだが、話者が4人なら `left2` を左右反転した位置に
   4人目の枠が自動で足される（`update_template_layout` で `right2` を渡せば、その枠を足してから動かす）。
   **できないこと**: 同時に舞台にいない2人でも1つの枠を交代で使うことはできない（枠より多い話者は位置を書いても出ない）。
   位置を書かないキャラの立つ側（左・右）と向きは枠で決まる。4人の動画で同じ側の2人だけを舞台に残すと、
   2人とも同じ側に立つ（左右に分かれ直しはしない。分けたいなら `stage` に `x` を書く）。`stage` に話さないキャラは書けない。
   重なり順（どちらが手前か）は枠で決まり、行ごとには変えられない。退場・登場・移動の速さ（約0.3秒）、
   アイキャッチの長さ（1.8秒）と見た目、暗転の長さは変えられない。背景の動きは動画の背景には掛からない。
   `story-talk`（いらすとや）は話者だけを出すテンプレートなので `stage` は効かない（背景の動き・場面転換・
   アイキャッチは効く）。

   **声だけの話者（任意）: 動画の `voiceOnly`・行の `offstage`。** 利用者が「ナレーターは画面に出さないで」
   「電話の向こうの声にして」「画面の外から声だけ」と言ったら、テンプレートではなくこの2つを書く。
   どちらも書かなければ以前と同じ見た目のまま（台詞のある行の話者はその行で登場する）。
   - `voiceOnly`（`create_yukkuri_video` の最上位。`["<キャラの id>", ...]`）: そのキャラは**動画の最後まで一度も画面に出ない**。
     台詞の行でも舞台に立たず、テンプレートの枠も使わない（空いた枠はほかの話者が使う。例: `tohoku-ahiru-talk` で
     ずんだもんを声だけにすると、ずんだもんの枠につむぎが座れる）。その行で `cameraMode:"dynamic"` でも寄らない
     （作るときの自動の寄りも付かない）。**声と字幕は普段どおり**（字幕に話者名を出す設定なら名前も出る）。
     書けるのは**台本で話すキャラだけ**（それ以外は `400 validation_error`、`invalidVoiceOnly` に id）。
     あとから `update_project_settings` の `voiceOnly` で変えられる（`null` か `[]` で外す。音声は作り直さなくてよい）。
     今の値は `get_project` の `voiceOnly`（無ければ `[]`）。
   - `offstage: true`（行ごと。`create_yukkuri_video` / `update_lines`（`false`・`null` で外す）/ `insert_lines` / `split_line`）:
     **その行の話者は声だけで、その台詞では舞台に出てこない**（まだ出ていないキャラは出ないまま）。
     **既に舞台に居るキャラは残る**（口も動く）。姿を消して声だけにするなら、同じ行の `stage` から外す（その行で退場する）。
     その行だけに効き、引き継がない。`offstage` の行でしか話さず、どの行の `stage` にも書かないキャラは一度も画面に
     出ず、枠も使わない。`split_line` では元の行と同じ話者のパーツに引き継ぐ。`get_project` の行に `offstage: true` が出る。
     `yukkuri-talk` は `stage` を書かない動画では話者が最初から最後まで出ているので、`offstage` で変わるのは
     「`offstage` の行でしか話さないキャラが席に座らない」ことだけ（途中から出すなら `stage` を書く）。
   - 声だけの話者と一緒に**使えないもの**（`400 validation_error`）: `voiceOnly` のキャラを `stage` に書く
     （`voiceOnlyOnStage`）・`action` で動かす（`invalidAction`）・`camera` の `target` にする（`invalidCameraTarget`）。
     `offstage` の行の話者をその行の `action` で動かす（`target` を省いたものを含む）・その行の `camera` の `target` にする
     （`"speaker"` を含む）。`update_project_settings` で `voiceOnly` にしたキャラがそうした行に書かれていると 400 で、
     `conflictLines` に行番号が返る（`update_lines` で直してから）。
   - 声だけの行の `camera`: `target` を省くと寄る先は画面の真ん中（`zoom` を書かなければ寄らない）。揺れ（`shake`）は効く。
   - `story-talk`（いらすとや）では、その行の話者の絵を出さない（背景と字幕だけ）。
   - テンプレートの `characters[].voiceOnly`（`update_template_style`）は**枠**を声だけにするもので別物（そのテンプレートを使う
     全部の動画に効き、枠は空かない）。1本の動画のキャラを声だけにするなら動画の `voiceOnly`。
   - 話者の検査（テンプレートの一座。`cast_mismatch`）は声だけのキャラにも掛かる（`yukkuri-talk` の動画で東北勢を声だけに、はできない）。
   - 古い話者 id の `_narrator_hidden`・`narrator`（どちらも声はずんだもんの声（VOICEVOX 3）で固定され、選べない）は互換のために受け付けるが、
     **新しく書かない**。姿を出さない話者は、声を選べる普通のキャラ id で書いて `voiceOnly` に入れる。

   ```json
   {"title":"電話がかかってきた","voiceOnly":["metan"],"script":[
     {"speaker":"metan","text":"これは、ある夏の日のお話。"},
     {"speaker":"zundamon","text":"今日も暑いのだ。"},
     {"speaker":"anko","text":"もしもし、聞こえるのだ？","offstage":true},
     {"speaker":"zundamon","text":"あんこ！どこにいるのだ？"},
     {"speaker":"anko","text":"今そっちに行くのだ！","stage":["zundamon","anko"]}
   ]}
   ```
   めたん（ナレーター）は一度も画面に出ない。あんこは3行目では声だけで、5行目の `stage` で登場する。

   **キャラの動き（任意）: `action`。** 利用者が「驚いたところで跳ねさせて」「お礼でお辞儀させて」「ボケに
   ずっこけさせて」と言ったら、その行に `action` を書く。立ち絵の**全体**を跳ねさせる・揺らす・傾けるので、
   組み込み（東北・ゆっくり）・自作キャラ（PNG 1枚・PSD 取り込み）のどれにも効き、口パク・表情・ポーズ・
   まばたきは動きの間も続く。**その行の頭で1回だけ動き、引き継がない**。書かなければ以前と同じ見た目のまま。
   作ったあとも `update_lines` / `insert_lines` / `split_line` で足せる・外せる（`null`）。音声は消えない。
   - 書き方: `"action":"jump"`（**その行の話者**が動く）/ `"action":{"type":"fall","target":"metan"}`（**target のキャラ**が動く。
     話していない聞き手のリアクションに）/ 配列で複数のキャラを同時に（4つまで。同じキャラは最初の1つ）
     `"action":["jump",{"type":"bounce","target":"anko"}]`。
     target に書けるのは**台本で話して画面に出るキャラだけ**（ナレーター・台本に居ないキャラは `400 validation_error`、
     `invalidAction` に id）。

   | action | 動き | 長さ | 使いどころ |
   |---|---|---|---|
   | `jump` | その場で大きく跳ぶ（ためて縮む→伸びて跳ぶ→着地で縮む） | 約0.8秒 | 驚き・大喜び（`surprise`・`excited` と） |
   | `hop` | ぴょんぴょん小さく2回跳ねる。**同じ行の `stage` で立ち位置（`x`・`y`・`scale`）が変わるなら、すべって歩く代わりに跳ねながら移る**（1回約0.3秒で2〜4回。遠いほど多い） | 約0.6秒 | 近づく・駆け寄る・軽いうれしさ |
   | `shake` | 小刻みに震える | 約0.6秒 | 怒り・怖さ・寒さ（`angry`・`sad` と） |
   | `bow` | 顔の向いている方へお辞儀する | 約1秒 | お礼・お詫び・あいさつ |
   | `spin` | くるっと振り向く。同じ行の `stage` で向き（`facing`）が変わるなら前の向きから新しい向きへ、変わらなければ一回りして元の向きへ | 約0.5秒 | ハッとする・話を切り替える |
   | `fall` | ずっこける（後ろへ傾いて下へ沈み、しばらくそのまま、行の終わりまでに起き上がる） | 行の長さに合わせて1.3〜3秒 | ボケへのツッコミ・オチ（聞き手に） |
   | `bounce` | はしゃぐ（小刻みに3回弾みながら左右に揺れる） | 約0.75秒 | うれしい・はしゃぐ |

   動き出すのは行の音声の頭（アイキャッチのある行は明けてから、その行で登場するキャラは入りきってから）。
   **目安: 盛り上がり・驚き・オチ・あいさつに。毎行には付けない**（5〜10行に1回くらい。続けると落ち着かない）。
   `cameraMode:"dynamic"`・素材・4人の並び・`stage` の立ち位置・縦型と一緒に使える。`story-talk` では効かない。
   **できないこと**: 手足だけを動かす（腕は `pose`）、動きの大きさ・速さの指定、1行で同じキャラに2つの動き。
   ```json
   {"title":"コーヒーの値段","script":[
     {"speaker":"zundamon","text":"えっ、そんなに安いのだ！？","emotion":"surprise","action":"jump"},
     {"speaker":"metan","text":"近くで説明するわね。","stage":["zundamon",{"id":"metan","x":60}],"action":"hop"},
     {"speaker":"zundamon","text":"つまり全部ボクのおかげなのだ。","emotion":"smug"},
     {"speaker":"metan","text":"なんでそうなるのよ！","action":["shake",{"type":"fall","target":"zundamon"}]},
     {"speaker":"zundamon","text":"見てくれてありがとうなのだ。","action":["bow",{"type":"bow","target":"metan"}]}
   ]}
   ```
   **表情（`emotion`）**: 立ち絵の目・眉・口・顔色・汗や涙の差分を組み合わせた**16種**。どのキャラも全部描ける。
   人気の解説動画は**ほぼ毎行で表情が変わる**ので、セリフの気持ちに合わせて行ごとに変えること
   （同じ表情が3行以上続くと単調に見える）。
   下の16種の英語名か、日本語の表情名（「驚き」「ドヤ顔」など）で書く。**それ以外の名前は 400 `validation_error`**
   （作る・`update_lines`・`insert_lines`・`split_line` のどれでも同じ。以前は作るときだけ黙って捨て、直すときはそのまま
   保存していた）。普通の顔は書かないか `"normal"`。`pose` も同じく、知らない名前は 400。

   | emotion | 表情 | 使いどころ |
   |---|---|---|
   | `joy` | 喜び | うれしい・楽しい・納得して明るく |
   | `excited` | 大喜び | テンションが上がる・「やったー！」 |
   | `surprise` | 驚き | 「えっ！？」と意外な事実 |
   | `shocked` | ガーン | 青ざめる・大ショック・「そんな…」 |
   | `sad` | 悲しみ | しょんぼり・残念 |
   | `crying` | 泣き | 涙が出るほどつらい・感動 |
   | `angry` | 怒り | 許せない・ぷんぷん |
   | `troubled` | 困り・焦り | まずい・どうしよう・冷や汗 |
   | `thinking` | 考え中 | 「うーん」・疑問を口にする |
   | `confused` | 混乱 | 目が回る・わけが分からない |
   | `doubtful` | ジト目 | 疑う・呆れる・「本当に？」 |
   | `smug` | ドヤ顔 | 得意げ・「ふふん」 |
   | `embarrassed` | 照れ | 恥ずかしい・褒められて |
   | `relaxed` | ほっこり | 安心・なるほど・のんびり |
   | `explaining` | 説明 | 要点・理由・手順を話す |
   | `love` | 好き | 大好き・うっとり |

   **ポーズ（`pose`）**: 腕の形。表情とは別に選べる。ずんだもん・四国めたん・春日部つむぎ・あんこもんだけに効く
   （ゆっくりの霊夢・魔理沙は腕が無いので無視される。キャラに無いポーズも無視）。
   `point`（指さし・説明）/ `wave`（手を挙げる・あいさつ）/ `cheer`（両手を挙げる）/ `hip`（腰に手）/
   `think`（あごに手）/ `mouth_cover`（口元に手）/ `whisper`（ひそひそ）/ `shh`（しーっ・秘密）/
   `chest`（胸に手・不安）/ `mic`（マイク）/ `cross_arms`（腕組み・あんこもん）/ `peace`（ピース・つむぎ）/
   `hold`（手を組む・めたん）/ `normal`。省略すると表情ごとの既定の腕（説明なら指さし、怒りなら腰に手 など）。
   **キャラごとに描けるポーズ**（無いポーズは黙って無視される。`list_characters` の `poses` でも分かる）:

   | キャラ | 描けるポーズ |
   |---|---|
   | zundamon | normal, point, wave, cheer, hip, think, mouth_cover, whisper, chest, mic |
   | metan | normal, point, shh, whisper, mouth_cover, mic, hold |
   | tsumugi | normal, point, peace, cheer |
   | anko | normal, point, wave, cheer, hip, think, mouth_cover, shh, cross_arms, mic |
   | reimu・marisa | なし（腕が無い） |
   | 自作キャラ | `list_characters` の `poses`（設定したものだけ） |

   **効果音（`effect`）**: 例: `"ショック1"`, `"きらーん1"`, `"ひらめく1"`, `"チーン1"`（一覧は `list_bgm` の `effects`）。
   利用者が上げた効果音の url も書ける（「BGM と効果音」）。

   **`emotion`・`pose`・`effect`・`cameraMode` は、1行も書かなければセリフの手がかりから自動で付く**
   （`cameraMode` の自動は、どの行にも `cameraMode` も `camera` も書かなかったときだけ）
   （「うぅ」→泣き、「どうしよう」→困り、「一つ目」→指さし、「ここだけの話」→しーっ など）。
   1行でも書いた項目は、書いた指定だけが使われる（自動と混ぜない）。**`null` で「書いた」に数えられるのは
   `cameraMode` と `effect` だけ**（寄り・効果音を一切入れたくないときに使う）。`emotion`・`pose` に `null` は
   書けない（400）。表情・ポーズの自動を止めたいなら、どれか1行に値を書く。
   細かく演出したいときは全行に書くとよい。
   **自動で付くのは作るとき（`create_yukkuri_video`）の1回だけ。** 作った後に `update_lines` で変えた指定が、
   自動で上書き・付け直しされることはない。
5. **生成する**: `create_yukkuri_video`（または `POST /agent/generate`）に `{ title, backgroundImageUrl, script }` を渡す。
   既定で MP4 を焼く。試すだけなら `output: "draft"` で作って `create_preview_link`（0クレジット・全編・音つき）。
   MCP の応答には `editorUrl`（画面で台本を開く URL）が入る。下書き（`output: "draft"`）を作ったら、利用者にこの URL を伝えること——画面の一覧から探すより早い。
6. **受け取る**: `renderId` が返るので `get_render` でポーリングする
   （`jobId` が返ったときは、`get_job` が `succeeded` になってから `result.renderId` を使う）。
   **完了の判定は `done === true` かつ `outputFile` が非空**（下の「レンダーの成否判定」）。
   その `outputFile` をユーザーに渡す。`callbackUrl` を渡しておけば、完了時に
   こちらから通知する（そちらの本文は `downloadUrl`）。

## 直す（ユーザーと会話しながら）

このサービスに編集画面を開いてもらう必要はない。**ユーザーの「ここを変えて」を
そのまま受けて、あなたが直す。**

1. **読む**: `get_project` で今の台本を行番号つきで受け取る。何行目を直すのかを
   ユーザーの言葉から特定する。
   応答に `subtitleWarnings` が含まれる場合は、字幕が長すぎる行がある。
   `fits: "shrunk"` = フォントを自動縮小して表示（見栄えが下がる）、`fits: "overflow"` = 下限でも収まらず画面外にはみ出す。
   どちらも `suggestion` に書いてある文字数以内に収めるよう `update_lines` で分割か短縮すること。
2. **直す**: `update_lines` に**変える行だけ**を渡す。渡さなかった行は一切変わらない
   ので、ユーザーが気に入っている部分を壊す心配がない。
   ```json
   { "edits": [ { "index": 3, "text": "実は理由はもっと単純なのだ" },
                { "index": 5, "backgroundImageUrl": "https://example.com/bg2.jpg" } ] }
   ```
   行数を変えたい場合は専用ツールを使う:
   - **`insert_lines`**: `after` (挿入位置の前の行番号、-1 で先頭) と `lines` (行データ配列) を渡す。
     挿入後、後続の行番号がすべて再割り当てされる。`get_project` で最新の行番号を確認すること。
   - **`split_line`**: `index` (分割する行番号) と `parts` (分割後の各パーツ) を渡す。
     `materialImageUrl`/`materialBox`/`voice` は全パーツに引き継がれる。`chapter`/`effect`/`bgm`/`emphasis`/`card`/`photo`/
     `backgroundImageUrl`/`cameraMode`/`camera`/`stage`/`backgroundMotion`/`transition`/`eyecatch`/`action` は先頭パーツのみ。
     `offstage`（声だけの行）は元の行と同じ話者のパーツ全部に引き継ぐ。
   - **`delete_lines`**: `indexes` (削除する行番号の配列) を渡す。全行削除は不可。
     削除後、残った行の番号が再割り当てされる。
3. **音声を作り直す**: `text` / `speaker` / `reading` / `voice`（声の速さ・高さ）を変えた行は、**その行の音声が
   無効化される**（応答の `audioInvalidated` に行番号が入る。`text` のルビを直したときも同じ）。古い音声を残すと
   新しい字幕と違うことを喋る動画になるため、こちらで必ず消している。
   読み違いを直したときは、先に `check_readings` でその行の読みを確かめる。
   `generate_audio` を呼んで作り直す。辞書（`set_pronunciation` / `delete_pronunciation`）を変えたときは
   行の音声は消えないが、`get_project` の `staleAudio` に出る——`generate_audio` を引数なしで呼べば作り直される
   （自作キャラの既定の速さ・高さ `update_character` の `voice` を変えたときも同じ）。表情・ポーズ・素材・演出を変えても音声は消えない
   （ただし表情で声の高さが少し変わるので、合わせたいなら `generate_audio` に `force: true` とその行）。
4. **確かめる**: `create_preview_link` の共有リンクで、直した箇所を**0クレジット**・全編・音つきで見せる。
   返事の「次の一手」（MP4 の見積もり・残高・`render_mp4` か `create_checkout` か）も一緒に伝える。
5. **出す**: `render_mp4`（動画1分につき1クレジット・全編・本番解像度）。

台本を作り直す（`create_yukkuri_video` をもう一度呼ぶ）のは最後の手段。
ユーザーが良いと言った行まで消えるうえ、レンダーにもう一度課金される。

## 読みを確かめて直す（`check_readings` / ルビ / 辞書）

読み上げエンジンは漢字を読み違える（「今日」をきょう／こんにち、「方」をほう／かた、人名・地名・専門用語）。
声を聞くまで誰も気づけないので、**音声を作る前に `check_readings` を呼ぶ**（消費なし。音声は作らない）。

    check_readings { projectId }                 # lineIndexes を省くと本文のある全行（1回100行まで）

各行に、エンジンに渡す文（`spokenText`）と、**エンジンが実際に読む音のカタカナ（`kana`）**が返る。
`/` は句の区切り、`、` は間。VOICEVOX は長音を母音で書く（キョオワ＝きょうは、トオキョオ＝とうきょう）。
`applied` に、その行で効いたルビ・辞書の語・エンジンの組み込みの直し・`reading` を読んだかが入る。
`hasKanji: true` は、エンジンに渡す文にまだ漢字が残る行（読み違いが起きうる行）。100行を超える台本は
`remainingLineIndexes` を `lineIndexes` に渡して続きを確かめる。

`kana` を台本の意味と照らし、違う行を次のどれかで直してから `generate_audio` を呼ぶ。直したらもう一度確かめる。

1. **ルビ（その行のその語だけ）**: `update_lines` で `text` の語を `｜漢字《よみ》` にする。字幕は漢字のまま、声だけ読みになる。
   - `｜`（全角）でも `|`（半角）でもよい。`｜` から `《` までが親文字なので、漢字以外にも掛けられる（`｜AI《エーアイ》`）。
   - 短く `漢字《よみ》` とも書ける。**直前の漢字の並び全体**に掛かる（`東京都庁《とうきょうとちょう》` は4字すべて。
     一部だけなら `東京｜都庁《とちょう》`）。
   - 親文字・よみは50文字まで。改行と `《》｜` は中に入れない。完成していない記法（`《》` だけ、ひらがなの後の `《…》` 等）は
     そのまま字幕に出る。
   - 字幕・共有リンク・MP4 には親文字だけが出る。`get_project` は `text` をルビのまま返し、`subtitle` に字幕の文を添える。
2. **辞書（すべての動画の、その語すべて）**: `set_pronunciation { word, reading, note? }`。
   - 自分のすべてのプロジェクト・VOICEVOX と AquesTalk の両方で、**これから作る音声**に効く。長い語から先に当たる。
     ルビの中と、`reading` を書いた行には当たらない。
   - `word` は30文字まで（空白・改行・`《》｜` 不可。ひらがなだけの語は不可）。`reading` はひらがな・カタカナ・`ー` だけ（60文字まで）。
     1人500語まで（越えると `409 dictionary_full`）。一覧は `list_pronunciations`、消すのは `delete_pronunciation`。
   - 1文字の漢字のような短い語は、ほかの語の中でも置き換わる。登録したら `check_readings` で確かめる。
   - アクセント（高低）は指定できない。
   - **作り済みの音声には自動では反映されない。** 辞書を変えると、その語を含む行は `get_project` の `staleAudio`
     （行の `audioStale: true`）に出る。`generate_audio` を引数なしで呼ぶと、音声が無い行と一緒に作り直される。
     `render_mp4` はこの行を止めない（前の読みのまま焼ける）ので、焼く前に作り直すこと。
3. **reading（行全体）**: その行の読みをかなで全部書く。ルビで足りないときだけ。その行は辞書もルビも効かず、書いた読みをそのまま読む。

AquesTalk（ゆっくりの声）の `kana` は、エンジンの更新が済むまで `null`（`note` つき）になることがある。そのときは
`spokenText` に漢字が残る行を、ルビか辞書で読みを書いておくと確実。

## 声の速さ・高さ（行の `voice` / 動画全体の `voicePlaybackRate`）

速さと高さは2段で決まる。どちらも任意で、書かなければ今までどおりの声になる。

1. **行の `voice`**（その行だけ。次の行へ引き継がない）: `{"speed": 0.5〜2.0, "pitch": -0.15〜+0.15}`
   （速さ 1・高さ 0 が標準）。`create_yukkuri_video` の `script[]`・`update_lines`・`insert_lines`・`split_line`
   （REST も同じ）で書ける。読み上げエンジンが**その速さ・高さで合成し直す**（口パクもその声に合う）ので、
   書いた・変えた行は音声の作り直しが要る（`update_lines` の `audioInvalidated`。`generate_audio` で作り直す）。
   `update_lines` では渡した値に置き換え（書かなかった項目は標準）、`null` で外す。`split_line` は全パーツが引き継ぐ。
   範囲の外・知らない項目・空の `{}` は 400 `validation_error`。`get_project` は行の `voice` を返す。
   - 高さは、表情などで自動で付く小さな上下（±0.03。前からの規則）に足され、合わせて ±0.15 で止まる。
   - AquesTalk（ゆっくりの声）は同じ `pitch` でも変化が大きめ（0.03 ごとに AquesTalk の高さ 100 に対して ±10）。速さは
     AquesTalk の話速に掛ける。
   - 使いどころ: 驚き・興奮の早口 `{"speed":1.3,"pitch":0.05}` / 落ち込み・ささやき `{"speed":0.85,"pitch":-0.04}` /
     ゆっくり言い切る決めゼリフ `{"speed":0.8}`。毎行には書かない（目安: 10行に1〜2行）。
2. **自作キャラの既定**（`create_character` / `update_character` の `voice`。同じ形）: そのキャラの全行に効く。
   行の `voice` と合わせるときは、**速さは掛け算・高さは足し算**（合わせたあとも範囲で止まる）。既定を変えると、
   そのキャラの作り済みの音声は `get_project` の `staleAudio` に出て、`generate_audio`（引数なし）が作り直す。
3. **動画全体の `voicePlaybackRate`**（0.5〜2.0。**既定は 1.1**——何も指定しない動画は 1.1 倍で読み上げる）:
   焼くときと共有リンクで音声の再生に掛かる（音声の作り直しは要らない）。行の速さとは**掛け合わせ**になる
   （行の `speed` 1.2 なら 1.2×1.1 の速さで聞こえる）。
   - `create_yukkuri_video` の `voicePlaybackRate`・`update_project_settings`（`PATCH /api/v1/projects/{id}/settings`。
     `null` で既定に戻す）・`render_mp4` の `voicePlaybackRate` で決める。どれも**プロジェクトに残る**ので、共有リンクと
     あとの `render_mp4` も同じ速さになる（`render_mp4` で省略するとプロジェクトの値、無ければ 1.1）。
   - 今の値は `get_project` の `voicePlaybackRate`（既定のままなら `voicePlaybackRateIsDefault: true`）。
   - 焼く前に速さを確かめるなら、`update_project_settings` で変えて `create_preview_link` で聞く。

遅く読むほど動画は長くなり、MP4 のクレジット（動画1分につき1）も増える。`create_yukkuri_video`（mp4）の受け付けの
見積もりは、行の `speed` と `voicePlaybackRate` が標準より遅い分だけ伸ばす（速い分は縮めない。実際の長さで精算する）。

## MP4 は音声を自動生成しない

`render_mp4` は**行に載っている音声を再生するだけ**で、音声を作りはしない。
音声が無い行があるまま呼ぶと、**課金せずに `409 missing_audio`** を返して止まる
（無音の動画にクレジットを払わせないため）。応答の `missingAudioLines` に対象行が
入っているので、`generate_audio` を呼んでから再実行する。

`create_yukkuri_video` は**中で音声まで作る**ので、この手順は要らない。
`get_project` → `update_lines` / `insert_lines` / `split_line` / `delete_lines` で直したあと
`render_mp4` を呼ぶときだけ、自分で `generate_audio` を挟む必要がある。
`insert_lines` と `split_line` が返す `needsAudio` 配列に音声が必要な行番号が入っている。

## 最小例（REST）

```bash
curl -X POST https://app.yukkurigen.com/api/v1/agent/generate \
  -H "Authorization: Bearer $YUKKURIGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "身近な熱力学のはなし",
    "backgroundImageUrl": "https://example.com/bg.jpg",
    "script": [
      { "speaker": "reimu",  "text": "今日は熱力学第二法則を解説するわ" },
      { "speaker": "marisa", "text": "エントロピーってやつだな" },
      { "speaker": "reimu",  "text": "そう。散らかった部屋が勝手に片付かないのと同じ理屈よ", "cameraMode": "dynamic" }
    ]
  }'
```

## 立ち絵とBGM

立ち絵は**テンプレートが自動で置く**。キャラを選べば画面に出る——別途の指定は要らない。

BGM は既定のものが入る。行ごとに曲を変える・止める・利用者の曲を使うのは行の `bgm`（次の「BGM と効果音」）。
動画全体の BGM は `GET /api/v1/projects/{id}/bgm`（再生できる URL つき）で確かめ、
`DELETE /api/v1/projects/{id}/bgm` で外せる（既定の BGM に戻る）。このパスに `POST` は無い——曲を作り直すなら
`POST /api/v1/projects/{id}/bgm/generate`（2クレジット。その動画の BGM を**置き換える**）。
曲を作って素材として持っておく（動画の BGM は変えない・行ごとに使う・ファイルとして落とす）なら `generate_bgm`（次の節）。
音量はテンプレートの `bgm.volume`（`update_template_style`）か `render_mp4` の `bgmVolume`。

## BGM と効果音（行の `bgm` / `effect`・手元の音を上げる・ダッキング）

**行の `bgm`**（`create_yukkuri_video` の `script[]`・`update_lines`・`insert_lines`・`split_line`。REST も同じ）:
その行から流す BGM。**次に `bgm` を書いた行まで引き継ぐ**ので、場面が変わる行・章の頭にだけ書く。書ける値は3つ:

- 組み込みの曲の名前（`list_bgm` の `bgm[].name`。例 `"Stream"`・`"かえるのピアノ"`・`"境内"`。`path` でもよい）
- 利用者が上げた音の `url`（下の `create_audio_upload` → `confirm_audio_upload` / `upload_audio`。画面の素材管理で上げた BGM の url も可）
- `"none"`（その行から BGM を止める。次に曲を書いた行までは無音）

曲が変わる所は 0.7 秒のクロスフェードで繋ぐ。曲は最後まで行くと頭からループする。`update_lines` で `null` を書くと
その行の指定を外す（前の行の曲のまま）。どの行にも書かなければ今までどおり（動画全体の BGM、無ければ既定の曲）。
`get_project` は行の `bgm` を返す（組み込みは `/audio/material/…mp3` のパスで返る）。音声の作り直しは要らない。

```json
{ "edits": [ { "index": 0, "bgm": "かえるのピアノ" },
             { "index": 12, "bgm": "none" },
             { "index": 14, "bgm": "境内", "effect": "和太鼓でドドン" } ] }
```

**行の `effect`**: その行の頭で1回鳴る効果音。組み込みの名前（`list_bgm` の `effects`）か、利用者が上げた音の `url`。
最大2秒鳴らし、行が短ければ行の長さまで（終わりは短くフェード）。

**書ける URL は、利用者が上げた音だけ**（`users/{自分}/assets/bgm/` か `.../sfx/` を指す YukkuriGen の URL）。
外部の URL・他人の素材は 400 `validation_error`（`invalidAudio` にどの行のどの値か）。署名付き URL を渡されたら
期限の無い元の URL に直して保存する。焼く前に、もう無い音（素材管理で消した等）は外してから焼く（その行は鳴らない・
その区間の BGM は無音）——レンダーごと落ちることはない。

**手元の音を上げる**（画像の「手元の画像を渡す」と同じ流れ。mp3 / wav / ogg だけ。m4a・aac は mp3 に変換してから）:

```text
// シェルがあるとき 1) URL を出す（MCP: create_audio_upload / REST: POST /api/v1/assets/audio-uploads）
{ "fileName": "opening.mp3", "mimeType": "audio/mpeg", "fileSize": 2483112, "kind": "bgm" }
→ data.key・data.uploadUrl・data.headers（Content-Type）・data.examples（curl / PowerShell の例）・data.expiresAt

// 2) PUT する（画像と同じ。Content-Type は mimeType と同じ値、大きさは fileSize ちょうど）
curl -sS -f -X PUT -H "Content-Type: audio/mpeg" --upload-file "<FILE_PATH>" '<uploadUrl>'

// 3) 確定する（MCP: confirm_audio_upload / REST: POST /api/v1/assets/audio-uploads/confirm）
{ "key": "<1) の data.key>" }
→ data.url を行の bgm（kind:bgm）か effect（kind:sfx）に書く

// シェルが無いとき（MCP: upload_audio / REST: POST /api/v1/assets/audio-uploads/inline）
{ "dataBase64": "SUQzBAAAAAAA...", "kind": "sfx", "fileName": "pon.mp3" }
→ data.url
```

- `kind`: `bgm`（BGM。署名付き PUT は最大 30MB、base64 はデコード後 8MB）/ `sfx`（効果音。どちらも最大 5MB）。
  どちらで上げた url も行の `bgm` / `effect` のどちらにも書ける（大きさの上限が違うだけ）
- 形式は中身の先頭のバイトで確かめる（mp3 / wav / ogg）。申告（`mimeType`・Content-Type）と中身が違えば 400、
  置かれていなければ 404 `not_found`、他人の `key` は 403 `forbidden`。同じ `key` の確定は何度呼んでも1件
- スコープ・容量・レート上限・クレジット（0）は画像のアップロードと同じ（「手元の画像を渡す」の「共通」）
- base64 は引数として書き出すので、実際に渡せるのは小さい音（目安は数百KB まで。短い効果音向き）
- **上げた音を動画に使ってよいか（著作権・利用規約）は利用者の責任**。利用者が作った音か、利用条件を確かめた
  フリー素材を使うよう伝える。組み込みの曲・効果音はそのまま使ってよい
- 上げた音の大きさは自動では測らない（中くらいの大きさとして鳴らす）。大きすぎ・小さすぎなら BGM は
  `bgm.volume`・`bgmVolume` で合わせ、効果音は音のファイル側で直してから上げ直す

**AI に曲を作らせる**（`generate_bgm` / REST: `POST /api/v1/assets/bgm/generate`。2クレジット）:
ACE-Step（AI の作曲エンジン）で1曲作り、**利用者の BGM 素材として登録する**——`create_audio_upload` の `kind:bgm` で
上げた音と同じ場所・同じ扱いなので、`list_bgm` の `uploaded.bgm` に出て、`url` を行の `bgm` に書ける。

```text
// 1) 受け付ける（いつもジョブ。202）
{ "tags": "8-bit chiptune, energetic, fast tempo, game battle", "duration": 180, "name": "battle" }
→ { "jobId": "...", "status": "queued", "pollUrl": "/api/v1/jobs/..." }

// 2) get_job（GET /api/v1/jobs/{jobId}）でポーリング。succeeded の result:
{ "url": "https://...amazonaws.com/users/.../assets/bgm/gen_....mp3",   // 永続。行の bgm にそのまま書く
  "downloadUrl": "https://...?X-Amz-...",                               // 1時間で切れるダウンロード URL
  "downloadUrlExpiresAt": "...", "durationSec": 180, "assetId": "aud_...", "trackId": "...",
  "mimeType": "audio/mpeg", "projectId": null, "appliedToProject": false }
```

- `tags`: 曲の雰囲気を**英語のタグ**で（1〜500 文字）。歌詞は付けない（lyrics を空で作る）
- `duration`: 30〜240 秒の整数（REST の既定は 60）。用途ごとに1回ずつ呼ぶ（例: タイトル 120・戦闘 180・ジングル 30）
- `projectId` は任意。渡しても**その動画の BGM（`bgm_path`）は変えない**。`applyToProject: true` のときだけ動画全体の BGM にする
  （`projectId` が要る。無ければ 400）。行ごとに流すなら返った `url` を行の `bgm` に書く
- 1曲に数分かかる（しばらく使われていないとエンジンの起動でさらに数分）。`Idempotency-Key`（MCP は `idempotencyKey`）で
  呼び直しても二重に作らない（処理中・成功済みなら同じ jobId が返る）。**失敗したジョブの鍵は空く**ので、同じ鍵で
  投げ直せば新しいジョブになる（失敗の分は返金済み。返金がまだ済んでいない間だけは失敗したジョブがそのまま返る）
- `downloadUrl` は `get_job` を呼ぶたびに署名し直す（ジョブには保存しない）。切れたら `get_job` をもう一度呼ぶ。
  音そのものは素材として残る（`url`）
- 2クレジットは受け付けで押さえ、生成に失敗したら返す（`failed` の `error.code` に理由）。大きさはストレージの容量に数え、
  足りなければ `quota_exceeded` で失敗して返金。API キーは有料プランか API 連携ライセンスが要る（無料プランは 402 `plan_required`。
  `/projects/{id}/bgm/generate` と同じ門）。クレジット不足は 402 `insufficient_credits`。レート上限は他の生成と同じ
- **権利**: エンジンの ACE-Step はコード・モデルとも Apache License 2.0 で公開されている。作った曲が既存の曲に似ないこと・
  Content ID に当たらないことを YukkuriGen は確かめていない（ACE-Step のモデルの説明も、生成した曲が独自のものかを
  使う人が確かめるよう求めている）。公開・販売に使ってよいかの判断は利用者の責任

**ダッキング（話している間だけ BGM を下げる）**: 既定は**切**（今までの動画は何も変わらない）。
`update_template_style` の `{"bgm":{"ducking":true}}`（そのテンプレートの動画すべて）か、
`update_project_settings` の `{"bgmDucking":true}`（その動画だけ。テンプレートより優先。`null` で外すとテンプレートに従う）で入れる。

- 話している行（音声のある行）の間、BGM を約 **-10dB** 下げる。話し始めの **0.15 秒前**から下げ始めて話し始めには
  下がりきり、話し終わりから **0.4 秒**かけて戻す。行の間が約 0.55 秒以下なら下げたまま繋ぐ（行の継ぎ目で上下しない）
- 間・アイキャッチ・話の前後・無音の行では元の大きさに戻る。BGM をしっかり聞かせたい動画で `bgm.volume` を
  上げるとき（例 `{"bgm":{"volume":0.5,"ducking":true}}`）に一緒に使う
- 焼かずに `create_preview_link` で聞いて確かめられる。今の値は `get_project` の `bgmDucking`（`null` はテンプレートに従う）

## クレジットとエラー

- 消費: MP4 レンダー = 動画1分につき1（端数切り上げ）、BGM 生成 = 2（1曲。`generate_bgm` も同じ）。台本の作成・修正・音声生成・共有リンクは 0。
  `create_yukkuri_video`（mp4）は受け付けで台本の文字数（各行の `text` と `reading` の長いほう）から多めに見積もって押さえ、
  音声を作ったあと実際の長さで精算する。差額を引けなければ焼かずに `insufficient_credits` で失敗し、押さえた分は全額返る。
  月の枠はフリー0・スタンダード100・プロ300（1分の動画ならそれぞれ100本・300本）。
  台本はあなた（呼び出し側の AI）が書く。台本はサーバで生成しない（LLM は呼ばない。BGM 生成は音楽の生成モデルを使う）。
- 不足時: `code: "insufficient_credits"` / `code: "plan_required"`。**諦めずに購入まで案内すること**（下の「支払いが要るとき」）。
- 権限不足: `code: "insufficient_scope"`。キーのスコープを確認。
- 認証まわり: `code: "unauthenticated"`（401）。キーを付けていなければ `reason: "signup_required"`
  （MCP は `error.data.code`）と会員登録の `signupUrl` が返る——**ユーザーに無料の会員登録を案内し**、
  済んだら呼び直す。キーが無効/失効なら `invalid_credentials`（接続し直すか、キーを発行し直す）、
  `code: "forbidden"`（403、他人のプロジェクトなど対象への権限が無い）、
  `code: "not_found"`（404、projectId / renderId が無い）。
  いずれもクレジットは消費されない。
- `code: "cast_mismatch"`（400）: 一座をまたぐ話者、またはテンプレートで描けない話者。声は出るが画面に
  映らない動画になるため、**課金せずに**中断する。作るとき（`create_yukkuri_video`）・テンプレートを替えるとき
  （`set_project_template`）・話者を変えるとき（`update_lines`）に確かめる。`undrawableSpeakers` と `casts` を見て揃える。
- 台本を直すとき（`update_lines`）: `code: "line_not_found"`（404、その行番号が無い
  ——`get_project` で今の行番号を取り直す）、`code: "duplicate_index"`（400、同じ
  行番号を2回指定した——1回にまとめる）、`code: "unknown_speaker"`（400、知らない
  話者id——応答の `knownSpeakers` から選ぶ）。
- 購入導線（`purchaseUrl` を開いた先）: `code: "price_unavailable"`（503）。その商品が
  現在購入できない設定になっている。利用者に伝え、別のプランを案内するか時間を置く。
- MP4 を出すとき（`render_mp4`）: `code: "missing_audio"`（409）。音声が無い行が
  あるため**課金せずに**止めた。`missingAudioLines` の行を `generate_audio` で
  作ってから、もう一度 `render_mp4` を呼ぶ。
- 同時に走らせすぎ: `code: "too_many_concurrent_renders"`（429）。**課金されない。**
  同時に実行できるレンダーはプランごとに上限がある（無料1 / スタンダード3 / プロ10）。
  `running` と `limit` と `retryAfter` が返るので、進行中のものが終わってから投げ直す。
  大量に作るときは、投げっぱなしにせず `get_render` で終わりを見てから次を出すこと。
- コールバックを頼んだのに受け付けられないとき: `code: "callback_unavailable"`（503）。
  **課金されない。** `callbackUrl` を外して投げ直し、`get_render` のポーリングで
  完了を待つこと（通知が使えないだけで、レンダー自体は普通に走る）。
- 画像を上げるとき（`create_image_upload` / `upload_image`）: `code: "quota_exceeded"`（429）。
  プランのストレージ容量が足りない（`current` と `limit` が付く）。要らない素材を消すか、プランを上げる。
- 音声生成の月の文字数（`generate_audio` / `create_yukkuri_video` の合成。読み仮名があればその文字数で数える）:
  無料プランで購入なし **10,000字**、**クレジットパックの残高がある・API連携ライセンスを持つ人は 500,000字**
  （無料プランのままでも）、スタンダード 500,000字、プロ 2,000,000字、エンタープライズ 5,000,000字（無制限のプランの暴走止め）。
  使い切ると `code: "tts_quota_exceeded"`（429）と `limit` / `used` / **`resetAt`**（戻る日時。購読者は次の請求日、
  それ以外は日本時間の月初）が返る。**`rate_limited` と違い、待っても `resetAt` までは戻らない**——
  投げ直さず、利用者に上限と戻る日を伝え、増やしたいなら `create_checkout` で standard かクレジットパックを案内する。
  一括の生成では残りの文字数で作れる行だけ作り、越えた行は結果の `error` に理由が入る（作れなかった行は課金も消費もない）。
- サーバ側の失敗（500）: `code` で分岐すること。

## 支払いが要るとき（402 を受けたら）

**402 で会話を終わらせないこと。** ユーザーはブラウザを見ていないので、
あなたが決済リンクを渡さないと先に進めない。

402 の応答には**買うべきものが1つだけ**入っている。選択肢を並べる必要はない。

```json
{ "code": "insufficient_credits", "shortfall": 3,
  "purchase": { "priceKey": "credit_pack_50", "label": "クレジットパック 50",
                "amountJpy": 2500, "credits": 50,
                "reason": "クレジットが 3 足りません。このパックで足ります（購入分は期限なし）。" } }
```

手順:

1. `create_checkout` に `purchase.priceKey` をそのまま渡す（`POST /api/v1/billing/checkout-link`）
2. 返った `url` をユーザーに渡す。`purchase.label` と `amountJpy` を添えて、
   何をいくらで買うのかを先に伝えること
3. `get_checkout` に `sessionId` を渡して確認する
   （`GET /api/v1/billing/checkout-link/{sessionId}`）
4. `status: "paid"` になったら、**元の操作を同じ `Idempotency-Key` で再試行する**
   （402 は鍵を解放するので、同じ鍵で通る）
5. 15分ほど `pending` のままなら打ち切り、「支払いが確認できませんでした」と伝えて止まる。
   **無限に待たないこと**

`plan_required`（無料プランで MP4。購入したクレジットも残っていない）のときは、`purchase` に**プラン**
（`standard`・月額）が、`alternative` に**いちばん小さいクレジットパック**（`credit_pack_50`・買い切り）が入っている。
**クレジットパックは無料プランのままでも買え、買ったクレジットが残っている間は MP4 を書き出せる**
（動画1分につき1クレジット・期限なし）。まず `purchase` を案内し、利用者が月額を望まなければ
`alternative.priceKey` を `create_checkout` に渡す。

BGM 生成などを API キー・MCP から呼んで `402 plan_required` が返ったときは、
`alternative` は付かない——パックでは開かないので、`purchase.priceKey`（`standard`）で案内する。

すでに有料プランを契約中に、もう一度サブスク（`standard` / `pro`）を申し込むと
`409 already_subscribed` が返る（購読が2本になり二重に請求されるのを防ぐため）。
プランの変更・解約は、応答の `manageUrl`（請求ページ）からカスタマーポータルを開いて**利用者本人に**
行ってもらう。

コンビニ・銀行振込などの後払いの決済は、**入金が確定するまで** `get_checkout` が `pending` のままで、
クレジットや権利も入金まで付かない。15分で `paid` にならなければ、「入金が確認でき次第反映されます」と伝えて止まる。

`get_checkout` は支払いの有無を直接返す。**残高の増減から推測しないこと**——
残高は月次リセットや他の操作でも動く。
  - `generate_failed` / `render_start_failed` … そのまま再試行してよい。
    予約したクレジットは返される。ただし返金はサーバ側の後処理なので、
    **応答を受け取った時点の残高には反映されていないことがある**
    （その場での返金が落ちた場合は、後追いの掃除処理が拾う。返金と、
    その記録の**両方**の書き込みが落ちた場合はサーバログにのみ残るので、
    残高が戻らないときは問い合わせること）。
    残高を当てにする処理を続ける場合は、少し置いてから
    `GET /api/v1/credits` で確認すること。
  - `progress_unavailable` … 進捗が読めなかっただけ。レンダーは継続している可能性があるので
    打ち切らず再試行する。
  - `internal_error` … 一覧取得など、課金を伴わない読み取りが失敗した。
    クレジットは動かない。そのまま再試行してよい。
- キャラid の打ち間違い: 未知のidは課金前に400になる（黙って別キャラや立ち絵なしで出力して
  クレジットだけ消費する、ということはしない）。どちらも**組み込みキャラと本人の自作キャラ**を受ける:
  - `create_yukkuri_video`（`POST /agent/generate`）: `code: "validation_error"` と `validSpeakers`（組み込みキャラの id）
  - `update_lines`（`PATCH /projects/{id}/lines`）: `code: "unknown_speaker"` と `knownSpeakers`（組み込みキャラの id）
  自作キャラの id は `list_characters` の `custom: true` から取る（上の一覧には入っていない）。
  話者を変えると、変えたあとの話者全員をそのプロジェクトのテンプレートで描けるかも確かめる（描けなければ `cast_mismatch`）。
- レート上限: `code: "rate_limited"` と **`retryAfter`（秒）**、同値の `Retry-After` ヘッダ。その秒数だけ待って再試行する。
- **200 が返っても完了とは限らない**: `render_mp4` の 200 は**開始**の応答で、
  常に `status: "rendering"` と `renderId` を返す。`waitForCompletion` は
  非推奨で**無視される**（付けても待たず、エラーにもならない）。
  課金は正当で、レンダーは走っている——進捗（`get_render` /
  `GET .../render/{renderId}/progress`）で完了を追うこと。
  **開始の応答に `outputUrl` は入らない。出力URLは進捗の `outputFile` から取ること。**
- レンダーの成否判定: 進捗（`get_render` / `GET .../render/{renderId}/progress`）は
  **`done` だけ見ると誤る**。
  - 成功: `done === true` **かつ** `outputFile` が非空
  - 失敗: `fatalErrorEncountered === true`、または `done === true` なのに
    `outputFile` が空（チャンク欠落、および後追いで失敗が確定した分）
  **失敗でも `done` が立つことがある**ので、必ず失敗判定を先に行うこと。
  どちらの失敗でもクレジットは自動返金される。`errors` は原因の配列だが
  リトライ予定のものも含むので、判定には `fatalErrorEncountered` を使うこと。
- **打ち切りの目安**: Lambda 側が強制終了されると進捗が固まり、`done` も
  `fatalErrorEncountered` も永久に立たないことがある。`overallProgress` が
  20分以上まったく動かない場合は失敗として扱い、ポーリングを止めてよい。
  失敗が確定した分のクレジットは返金される。ただし返金はサーバ側の後処理で
  行われるため、**打ち切った直後の残高には反映されていない**（進捗が
  失敗を返した場合はその時点、進捗が凍ったまま止まった場合は後追いの
  掃除処理の後）。残高は少し置いてから `GET /api/v1/credits` で確認すること。
  なお、打ち切ったあとにサーバ側が**動画は出来ていた**と判定して完了に
  変わることがある（その場合は課金されたまま）。打ち切りは「もう待たない」
  という意味であって最終結果ではないので、後で `get_render` を一度
  確認するか、`GET /api/v1/projects/renders` で結果を拾い直すとよい。

## 再投入しても二重課金しない方法

課金される操作（`create_yukkuri_video` / `render_mp4`）には冪等キーを付けられる。
**ヘッダ（`Idempotency-Key`）でも本文（`idempotencyKey`）でもよい**——MCP の引数名で
本文に入れても効く。レンダーは一番高い操作なので、特に付けること。同じ鍵で
再投入すると、最初に成功したときの応答がそのまま返り、課金は起きない。有効期間は24時間。
ジョブで受け付けた（202）ときは、ジョブが処理中・成功済みの間、同じ鍵で**同じ `jobId`** が返る。
同期とジョブは同じ鍵を共有する（片方で使った鍵をもう片方で使っても、二重には作られない）。

（2026-09-09 まで、REST を直接叩いて**本文に**入れた鍵は黙って捨てられていた。
応答が遅くて投げ直すと、同じ鍵なのにプロジェクトが2本でき、2回課金された。）

応答を落とした（タイムアウト、接続断、プロセス再起動）ときは、**同じ鍵で
そのまま投げ直せばよい**。鍵を変えると別の操作として扱われ、もう一度課金される。

鍵を控えていない場合は `list_projects` でプロジェクトを見つけ、
`GET /api/v1/projects/renders`（`?projectId=` で絞れる）でそのレンダーの
`renderId` を拾い直してから `get_render` で出力URLを得る。
返金済みのものは `status: "failed"` で返る。

同じ鍵の処理がまだ走っている間に投げ直すと `409` と `code: "in_flight"` が返る。
少し待って同じ鍵で再試行すること（別の鍵に変えると二重に課金される）。

**どの場合も、鍵は新しくしないこと。** 新しい鍵にすると、実は最初の呼び出しが
通っていた場合に二重課金になる。応答ごとの扱いは次のとおり:

| 応答 | 鍵の状態 | 次にすること |
|---|---|---|
| `202`（ジョブを受け付けた） | ジョブが持つ（処理中・成功済みの間） | 投げ直しても**同じ jobId** が返り、課金は起きない。`get_job` で追う。ジョブが `failed` になると鍵は空くので、理由を直してから**同じ鍵**でやり直す（レンダーの起動後に失敗したジョブだけは鍵が空かず、投げ直すと起動済みのレンダーの `renderId` が 200 で返る） |
| `402` insufficient_credits / plan_required | 解放される | 購入・アップグレード後、**同じ鍵**でそのまま投げ直す |
| `409` in_flight | 別のリクエストが保持中（その鍵のジョブが処理中の間は、`Prefer` なしの同期の呼び出しにもこれが返る） | 少し待って**同じ鍵**で再試行する |
| `503` unavailable（判定できなかった） | 取られていない | **同じ鍵**でそのまま再試行してよい。課金は起きていない |
| `503` unavailable（クレジット確保に失敗） | 保持されたまま | **課金されたか確定していない**。`GET /api/v1/credits` で残高を確認し、しばらく置いてから同じ鍵で再試行する（鍵は65分で自然に解ける） |
| `503` unavailable（ジョブの受け付けで `jobId` 付き） | 受け付けが入っていればジョブが持つ | 受け付けが入ったか分からなかった場合と、ジョブを積めなかった場合の2通りがあり、本文では区別できない。`GET /api/v1/jobs/{jobId}` で確かめる: ジョブがあって `queued` / `running` / `succeeded` なら課金されて進んでいる、`not_found` なら課金されていない、`failed` なら返金される。**同じ鍵**で投げ直してもよい: ジョブが `queued` / `running` / `succeeded` なら同じ jobId が返る（二重には課金されない）。`failed` なら鍵は空いていて、投げ直しは新しい jobId の新しいジョブとして改めて課金されるが、先に押さえた分は返金されている（二重にはならない）。`not_found` なら新しく受け付ける。例外は、レンダーを起動したあとにジョブだけが失敗した場合（まれ）で、返金されず鍵も空かない。同じ鍵で投げ直すと、起動済みのレンダー（`renderId: "agent-{jobId}"`）が 200 で返る（2本目は作られない） |
| `500` `render_start_failed` / `generate_failed` | 解放される | そのまま同じ鍵で再試行してよい。課金分は返金済みか、返金待ちとして記録済み（両方の書き込みが落ちた場合のみサーバログにのみ残る） |

クレジット確保の失敗だけ鍵を保持するのは、Firestore のトランザクションが
コミット後に失敗しうるためで、そこで解放すると再試行が二度目の課金になる。

失敗した応答は記録しないので、500 が返った場合の再試行は通常どおり実行される
（一時的な障害が鍵に焼き付いて恒久化することはない）。**逆に言えば、鍵は
「もう一度実行してよいか」を判定しない**——再実行してよいかは、鍵ではなく
応答の `code` を見て上の表で判断すること。

## 大量に作る

1本ずつ20往復する必要はない。**`create_yukkuri_videos_batch`（`POST
/api/v1/agent/generate/batch`）に最大20件まとめて渡せる。**

```json
{
  "idempotencyKeyPrefix": "run-2026-09-06-a",
  "items": [
    { "title": "1本目", "script": [{ "speaker": "reimu", "text": "ここが1本目です" }] },
    { "title": "2本目", "script": [{ "speaker": "marisa", "text": "ここが2本目だぜ" }] }
  ]
}
```

各要素は `create_yukkuri_video` と**まったく同じ形**。中では1件ずつ順に
処理され、認証・レート制限・クレジットはそれぞれに効く（まとめても
安くならない）。結果は `results` に1件ずつ index 順に並ぶ（同期なら 200 の応答そのもの、
ジョブで受け付けたなら `get_job` の `result`。下の「バッチもジョブで受け付ける」）:

- mp4 の件は `projectId` と `renderId`（ジョブで受け付けたなら、その件のジョブの
  `jobId` も）が入る。`renderId` は**レンダーの開始まで済んだ**という意味で、動画はまだ焼いている
  途中——`projectId` と一緒に `get_render` に渡して完成を待つ。draft の件は `renderId` が無い。
- `status` はその件の HTTP ステータス（整数。単発の `create_yukkuri_video` を呼んだときと同じ）。
- 途中の1件が失敗しても、**成功した分の `projectId` は必ず返る**。捨てないこと。
- 残高切れ（`insufficient_credits` / `plan_required`）とレート制限
  （`rate_limited`）を受けた時点で**そこで打ち切る**。`stoppedAtIndex` /
  `stopReason` / `notAttempted` が付く。
- **`notAttempted` は「失敗した」ではなく「まだ作っていない」。** 失敗と
  混同して作り直すと、成功した分をもう一度作って二重に払うことになる。
  投げ直し方は下の2つだけ（`nextStep` にも同じことが入る）。

`idempotencyKeyPrefix` を付けると、各件へ `<prefix>:<index>` が冪等キーとして
渡る。打ち切られたあとの投げ直しは、次の**どちらか**にする:

1. **全件を同じ `idempotencyKeyPrefix` でそのまま送り直す。** 作った件は同じ結果が返り、二重課金されない。
   ただし冪等キーが効くのは**その件が終わってから 24 時間以内**だけ。過ぎた件は同じ prefix でも
   もう一度作られて課金される。prefix を付けていなかったなら、この方法は使えない。
2. **打ち切った件（`stoppedAtIndex`）と `notAttempted` の分だけを、新しい `idempotencyKeyPrefix` の新しい
   バッチで送る。** 24 時間を過ぎた・prefix を付けていなかったときはこちら。

**同じ prefix のまま一部の件だけを送らないこと。** 番号がずれて別の件の冪等キー（`<prefix>:<index>`）に
当たり、冪等キーは本文を比べないので、その件の前の結果が成功として返るだけで、足りない動画は作られない。

レート制限は1分あたりの呼び出し回数（無料5 / スタンダード10 / プロ20）で、
バッチの中の1件も1回と数える。無料プランで20件投げると6件目で打ち切られる
——これは仕様どおりで、1分後に上の2つのどちらか（全件を同じ prefix で、または打ち切った件と
`notAttempted` の分だけを新しい `idempotencyKeyPrefix` の新しいバッチで）で投げ直せばよい。

### バッチもジョブで受け付ける

20件を1回の応答の中で順に作ると、接続が切れるまでに終わらない。
`Prefer: respond-async` を付けると（`Authorization: Bearer` のときだけ）、受け付けだけをして
数秒で 202 を返し、各件はサーバ側のジョブが1件ずつ順に作る（サーバの設定によっては付けても
ジョブにならず、同期で処理して 200 を返す）。**MCP の `create_yukkuri_videos_batch` は自分でこれを
付ける**ので、MCP から使うときは何もしなくてよい。

```json
{ "jobId": "...", "status": "queued", "pollUrl": "/api/v1/jobs/{jobId}", "requested": 2 }
```

流れ（MCP）:

    create_yukkuri_videos_batch { idempotencyKeyPrefix, items }   # 数秒で jobId が返る
    get_job { jobId }                                             # succeeded まで待つ（1件ずつ作る）
    get_render { projectId, renderId }                            # result.results の各件ごとに

- **202 には `results` が入らない。** まだ1本も作っていない。クレジットも押さえておらず、
  各件を作り始めるときにその件の分を押さえる。
- 受け付けでその場に返るもの（何も作られない）: 認証（401）、1件目のスコープ不足と、どの件に
  要るスコープも持っていない場合（403 `insufficient_scope`。mp4 は `render:mp4` と
  `audio:generate`、draft は `projects:write`）、本文の合計が 4 MiB を超える（400
  `validation_error`。1件で 512 KiB を超える件は、その件だけが結果で 400 になる）、レート上限
  （429 `rate_limited`。受け付け1回ごとに数える）、進行中（queued / running）のバッチが同じ
  利用者にすでに2件ある（429 `rate_limited`。`Retry-After` は 300 秒。本文の `activeJobIds`
  （`activeJobs` に `jobId` と `pollUrl`）が進行中のバッチなので、投げ直し続けずに `get_job` で追い、
  どちらかが終わってから投げ直す。受け付けの 202 を取り落としたときも、ここで `jobId` が分かる）。
- `succeeded` の `result` は同期の 200 と同じ封筒で、打ち切ったなら `stoppedAtIndex` /
  `stopReason` / `notAttempted` も入る。MCP の `get_job` は本文で各件を一覧にする。
- 1件のジョブが3時間待っても終わらないと、その件の結果は `child_timeout`（504）になり、次の件へ
  進む。**そのジョブは止めていない**ので、作り直さず、結果の `jobId` を `get_job` に渡して追う。
- サーバの設定によっては、進行中のバッチが次の件を作る前に打ち切られる（`stopReason` が
  `jobs_paused`、その件の結果は 503）。その件のジョブは作っておらずクレジットも押さえていないので、
  ほかの打ち切りと同じく、しばらく置いてから、全件を同じ `idempotencyKeyPrefix` で送り直すか、その件と
  `notAttempted` の分だけを新しい `idempotencyKeyPrefix` の新しいバッチで送る。
- **バッチのジョブが `failed` になっても返金ではない。** バッチのジョブ自体はクレジットを
  持たず、作り始めた件はそれぞれのジョブで課金されたまま進み、動画も作られる。
  `GET /api/v1/jobs/{jobId}` は失敗したバッチに `partialResults` を付け、作り始めた件（`started`。
  `projectId` / `renderId` / `jobId`）、ジョブのある失敗した件（`failedWithJob`。課金の扱いはその
  `jobId` で確かめる）、作られていない件（`notCreated`）、同じ冪等キーの処理が別の呼び出しで進行中
  だった件（`inFlight`。409 `in_flight`）、まだ作っていない件（`notAttempted`）と、投げ直してよい件の
  番号（`resendIndexes`）を分けて返す。MCP の `get_job` は本文でも同じ分け方で
  案内する。**`started` と `failedWithJob` の件は投げ直さない。** ただし `failedWithJob` のうち、その件の
  ジョブが失敗して返金された件は冪等キーが空いているので、同じ `idempotencyKeyPrefix` で全件を投げ直すと
  作り直され、1回課金される（投げ直す前に、その `jobId` を `get_job` で確かめる。`child_timeout` で
  まだ動いている件は作り直されない）。`progress.done` は動いている件を
  数えないので、この判断に使わないこと。`partialResults` が `null` なら件を読めなかったので、
  しばらく置いてから読み直す。
- 投げ直すときは、**`resendIndexes` の件だけを新しい `idempotencyKeyPrefix` の新しいバッチ**で送る
  （同じ prefix で一部だけを送ると、番号がずれて別の件の冪等キーに当たる）。同じ
  `idempotencyKeyPrefix` で全件を投げ直しても作り始めた件が二重に作られないのは、**その件のジョブが
  終わってから 24 時間以内**だけ（過ぎた件はもう一度作られ、課金される）。その期限は
  `partialResults.fullResendSafeUntil` に入る（`null` なら全件を投げ直さない）。
- **`inFlight` の件は `resendIndexes` に入らない。新しい `idempotencyKeyPrefix` のバッチに入れないこと**
  （新しい冪等キーは生きている予約を素通りし、別の呼び出しが作っている動画をもう一度作って払う）。
  しばらく置いてから、その件だけを同じ冪等キー（`<idempotencyKeyPrefix>:<index>`。prefix を付けて
  いなければその件の `idempotencyKey`）で送り直す。まだ進行中なら作らずに断られ、作り終わっていれば
  同じ結果が返る。
- `Prefer` を付けない REST の呼び出しは同期のまま。ただしサーバの設定によっては、同じ受け付けの
  あと一定時間だけ完了を待ち、終わらなければ `Preference-Applied` なしの 202 を返す。
  **202 が返りうるものとして扱うこと。**

## 完了を待たずに済ませる（コールバック）

MP4 レンダーは数分かかる。`render_mp4` に `callbackUrl` を付けると、
**完了/失敗したときにこちらから POST する**ので、進捗を見るためだけに
起きている必要がなくなる。

`render_mp4` の引数に `"callbackUrl": "https://example.com/hooks/yukkurigen"`
を足すだけでよい。

- `callbackUrl` は **https のみ・内部アドレス不可**。使えない URL は
  **課金せずに 400** で断る（黙って無視はしない）。
- 開始の応答に `callbackSecret` が1度だけ入る（利用者ごとに固定）。
  これを保存して署名を検証すること。

届く本文:

```json
{
  "event": "render.completed",
  "projectId": "PROJECT_ID",
  "renderId": "RENDER_ID",
  "status": "completed",
  "downloadUrl": "https://...(署名付き・6時間有効)",
  "occurredAt": "2026-09-06T00:00:00.000Z"
}
```

検証の手順（`X-Yukkurigen-Signature: t=<unix秒>,v1=<hex>`）:

1. `t` が現在時刻から**5分以上離れていたら捨てる**。これをやらないと、
   一度盗まれた本文と署名の組を永久に使い回される。
2. `HMAC-SHA256(callbackSecret, "<t>.<受け取った本文そのまま>")` を計算し、
   `v1` と**定数時間で**比較する。本文は再シリアライズせず、生のまま使うこと。

注意:

- 通知は**1回しか送らない**（届いたことは確認しない）。受け取り損ねたときは
  `GET /api/v1/projects/renders`（REST のみ。MCP ツールは無い）で拾い直せる。
- 転送（3xx）には従わない。宛先はリダイレクトさせず直接受けること。
- 通知が送れない状態のときは `code: "callback_unavailable"`（503、課金なし）。
  `callbackUrl` を外して投げ直し、`get_render` で待つこと。
- **通知の遅さについて（未確認の点あり）**: 完了は Lambda ではなく、こちら側が
  「終わったこと」を確定させた時点で送る。誰かが進捗を見ていれば即座だが、
  誰も見ていない場合は掃除の巡回で拾う。**巡回は10分おき**なので、
  最悪でも10分ちょっとで届く。急ぐ場合は `get_render` を1回叩けば
  その場で確定する（叩いた側の分はその時点で片付く）。

## 調整できること

台本以外の指定はすべて任意で、省略すれば既定値になる。openapi.json に
全項目があるが、よく使うものは:

- `platform`: 縦横と用途（youtube / tiktok / shorts）。向きはこれだけで決まる。省略するとプロジェクトに残した向き
  （`outputFormat` は `platform` を省き、プロジェクトにも向きが無いときの手がかりにしか使わない）。
- `fontSize` / `titleFontSize`: 字幕とタイトルの文字サイズ。
- `voicePlaybackRate`: 動画全体の読み上げの再生速度（0.5〜2.0。**既定 1.1**）。渡すとプロジェクトに残る
  （共有リンク・次の `render_mp4` も同じ速さ）。焼かずに変えるなら `update_project_settings`。行ごとの速さ・高さは
  行の `voice`（「声の速さ・高さ」）。
- `bgmVolume`: BGM 音量（0〜1）。話している間だけ下げるのはテンプレートの `bgm.ducking` か `update_project_settings` の `bgmDucking`。
- `materialImageFilter`: 素材画像のフィルタ（`blur` / `top-dark`）。
- `showOutroCard`: 終了カードの有無。

MCP のツール定義と openapi は同じ集合を公開している。片方にしか無い
項目があれば、それは不具合として扱ってよい。

## 画像を取り込む（`import_images` / `POST /api/v1/assets/import`）

画像生成サービスの出力 URL など、**期限付きまたは再配布できない URL** を動画に使いたいときに
YukkuriGen のアセットストレージ（S3）へ永続コピーし、materialImageUrl / backgroundImageUrl に
直接書ける永続 https URL を返す。期限のない公開 https URL はそのまま書いてよい（取り込み不要）。

```text
// MCP
{ "urls": ["https://dalle-output.example.com/tmp/abc123.png"], "type": "material" }

// REST
POST /api/v1/assets/import
{ "urls": ["https://..."], "type": "background" }
```

- **形式**: png / jpeg / webp・1件最大 10MB
- **件数**: 最大 20件/回
- **スコープ**: `projects:write`
- **クレジット消費**: 0
- 失敗した URL は他に影響しない（結果は件ごとに `ok` / `error` で返る）
- 成功した `url` を行の `materialImageUrl` または動画全体の `backgroundImageUrl` に渡す
- **利用者の PC にあるファイルは渡せない**（サーバが取りに行ける URL ではないため）。下の「手元の画像を渡す」を使う

## 手元の画像を渡す（`create_image_upload` → `confirm_image_upload` / `upload_image`）

利用者の PC にある画像（ComfyUI などで作った画像・スクショ・写真）を上げて、`import_images` と同じ扱いの
永続 URL（`materialImageUrl` / `backgroundImageUrl` にそのまま書ける）を受け取る。どれを使うかは環境で決める:

- **シェル（curl / PowerShell）を実行できる**（Claude Code・Codex CLI・Cursor など）→ `create_image_upload` で
  URL を出し、自分のシェルで PUT し、`confirm_image_upload` で確定する。最大 10MB
- **シェルが無いが、ファイルの中身は読める**（Claude Desktop にファイルを読む MCP だけがある、など）→ `upload_image` に
  base64 で渡す。上限はデコード後 8MB だが、base64 は引数として書き出すので**実際に渡せるのは小さい画像**（目安は数百KB まで）
- **画像が https の URL にある** → `import_images`

### シェルがあるとき（署名付き PUT）

```text
// 1) URL を出す（MCP: create_image_upload / REST: POST /api/v1/assets/image-uploads）
{ "fileName": "scene1.png", "mimeType": "image/png", "fileSize": 482113, "kind": "material" }
→ data.key・data.uploadUrl・data.headers（Content-Type）・data.examples（curl / PowerShell の例）・data.expiresAt

// 2) PUT する（<FILE_PATH> を手元のパスに、<uploadUrl> を返った値に置き換える）
curl -sS -f -X PUT -H "Content-Type: image/png" --upload-file "<FILE_PATH>" '<uploadUrl>'
Invoke-WebRequest -UseBasicParsing -Method Put -ContentType 'image/png' -InFile '<FILE_PATH>' -Uri '<uploadUrl>'

// 3) 確定する（MCP: confirm_image_upload / REST: POST /api/v1/assets/image-uploads/confirm）
{ "key": "<1) の data.key>" }
→ data.url を行の materialImageUrl か backgroundImageUrl に書く
```

- `fileSize` はファイルのバイト数**ちょうど**（`wc -c < <FILE_PATH>` / PowerShell は `(Get-Item <FILE_PATH>).Length`）。
  Content-Type と大きさは署名に入っているので、どちらかが違うと S3 が 403 で断る
- Windows PowerShell 5.1 の `curl` は Invoke-WebRequest の別名。curl の例を使うなら `curl.exe` と書く
- URL の期限は1時間。切れたら 1) からやり直す。**PUT しただけでは使えない**（確定まで呼ぶ）
- 確定では置かれた中身を確かめる（png / jpeg / webp か・先頭のバイトが Content-Type と一致するか・10MB 以下か）。
  違えばそのファイルを消して 400 `validation_error`（正しい `mimeType` で 1) から）。置かれていなければ 404 `not_found`
  （PUT がまだ・失敗した・期限切れ）。他人の `key` は 403 `forbidden`
- 同じ `key` の確定をもう一度呼んでも登録は1件（同じ `url` と `alreadyRegistered: true` が返る）

### シェルが無いとき（base64）

```text
// MCP: upload_image / REST: POST /api/v1/assets/image-uploads/inline
{ "dataBase64": "iVBORw0KGgo...", "fileName": "scene1.png", "kind": "background" }
→ data.url
```

- 形式は中身の先頭のバイトで決める（png / jpeg / webp だけ）。`mimeType` は省略できる（書くなら中身と一致させる。違えば 400）
- `data:image/png;base64,` の前置きや改行は付いていてもよい。欠けた base64 は読めずに 400 `validation_error`
- 会話に貼られた画像を「見た」だけでは中身の base64 は手に入らない。ファイルとして読めるときだけ使える

### 共通

- **スコープ**: `projects:write`（OAuth でつないだ鍵にも付いている）
- **クレジット消費**: 0
- **容量**: 大きさはプランのストレージ容量（無料 100MB・スタンダード 1GB・プロ 5GB）に数える。`create_image_upload` は
  URL を出した時点で押さえる（確定しなければ1時間で外れる）。足りなければ 429 `quota_exceeded`
- **レート上限**: 素材のアップロード用の枠（1分あたり 無料30・スタンダード60・プロ120 回）。URL の発行と確定はそれぞれ1回と数える。超えると 429 `rate_limited`
- `kind`: `material`（行ごとの素材画像、既定）/ `background`（動画全体・場面の背景）。どちらで上げた url も
  `materialImageUrl` / `backgroundImageUrl` のどちらにも書ける（分類が違うだけ）

## 自分のキャラを作る（`create_character` / `update_character` / `list_voices`）

利用者の「このいらすとやの PNG を、霊夢の声で喋らせて」は、あなたが画像1枚から話者を作って叶えられる。
できあがるのは、画面（「自分のキャラ」`https://app.yukkurigen.com/assets` の「画像を追加」→キャラ設定）で
作ったものと同じキャラで、`list_characters` に `custom: true` で出て、画面でも直せる・消せる。

```text
// 1) 画像を上げる（手元のファイル: upload_image か create_image_upload → confirm_image_upload / https の URL: import_images）
→ data.url（または data.assetId）

// 2) 作る（MCP: create_character / REST: POST /api/v1/agent/characters）
{ "name": "れいむ（いらすとや）", "imageUrl": "<1) の url>", "voiceOf": "reimu", "figure": "full" }
→ 201 characterId・kind（png）・imageUrl・voice（speakerId 1001・AquesTalk・霊夢（ゆっくり））・emotionImages・figure・face・defaultScale・templates

// 3) 話させる: characterId を create_yukkuri_video / update_lines の speaker に書く
{"speaker":"<characterId>","text":"今日はわたしが解説するよ。"}
```

- **画像**: `imageUrl`（`upload_image` / `confirm_image_upload` / `import_images` が返した url）か `imageAssetId`（同じく `assetId`）の
  どちらか1つ。**この利用者が YukkuriGen に上げた画像だけ**使える（ほかのサイトの URL は先に `import_images` で取り込む）。
  png / webp・10MB まで（立ち絵は背景が透明な画像を使う。jpeg は 400）。ほかの利用者の画像は 403 `forbidden`、
  ほかの URL・置かれていない・型と中身が違う画像は 400 `validation_error`
- **声**: `speakerId`（`list_voices` の値）か `voiceOf`（組み込みキャラと同じ声: `reimu` `marisa` `zundamon` `metan` `tsumugi` `anko`）の
  どちらか1つが必須。霊夢のゆっくり声は AquesTalk の `1001`、魔理沙は `1002`。`list_voices` の各声には `engine`・`label`・`gender`・
  `usedBy`（その声を使う組み込みキャラ）が付く
- **表情ごとの画像**（任意）: `emotionImages` に表情（`emotion` と同じ16種のキー）→ url。同じ構図・同じ大きさの差分の画像を使う。
  その表情の行はその画像で描かれ、付けない表情は近い表情の画像（例: `excited` → `joy`）か、無ければ通常の画像で描かれる。
  画面のキャラ設定の「感情別設定」と同じ項目
- **立ち絵の種類**（任意）: `figure` は `full`（全身。字幕の上に収まるよう小さめに置く）/ `bust`（上半身。画面の下端まで）。
  省略すると画面の下端まで使う
- **声の既定の速さ・高さ**（任意）: `voice` は `{"speed": 0.5〜2.0, "pitch": -0.15〜+0.15}`（行の `voice` と同じ形）。
  このキャラの全行に効き、行の `voice` とは速さが掛け算・高さが足し算で合わさる。`update_character` では渡した値に
  置き換え、`null` で外す。変えると作り済みの音声は `staleAudio` に出る（`generate_audio` を引数なしで呼ぶと作り直す）。
  `list_characters` / `GET /api/v1/agent/characters/{characterId}` の `voice` に `speed`・`pitch` として返る（標準なら無い）
- **顔の位置**（任意）: `face` は顔の中心を**立ち絵の絵の中の割合**で書く（`{"x":0.5,"y":0.2}`。左上が 0,0・右下が 1,1。
  目の上端と口の下端の真ん中あたり）。カメラが話者に寄る（行の `camera`・`cameraMode:"dynamic"`）ときに画面の真ん中に
  持ってくる所。**省略すると絵から推し量って保存し、返す**（肌の色の塊 → 首のくびれ → 絵の上端からの決まった割合の順）。
  寄った画で顔が真ん中より下に映ったら `y` を大きく、上なら小さく直す（`update_character`）。`null` で外すと、描くたびに絵から推し量る。
  寄らない動画の見た目には影響しない
- **大きさ**（任意）: `defaultScale`（0.5〜2。既定 1）。枠に合わせた大きさにこれを掛ける（足元を支点に。行の `stage` の `scale` とも掛け合わせる）。
  周りのキャラより小さく・大きく見える絵をそろえるのに使う
- **一座**: 東北勢と同じ（`tohoku-ahiru-talk`。ずんだもん等とは一緒に出せる）。声が霊夢でも、ゆっくり（霊夢・魔理沙）の立ち絵の動画には混ぜられない
- **容量**: 立ち絵の画像はキャラ用に複製し、その大きさをプランのストレージ容量に数える（足りなければ 429 `quota_exceeded`）。
  画面でキャラを消すと、その複製も消える（上げた元の画像は残る）
- **直す**: `update_character`（`PATCH /api/v1/agent/characters/{characterId}`）で `name`・声・`emotionImages`（url で設定、`null` で外す）・
  `figure`（`null` で外す）・`voice`（既定の速さ・高さ。`null` で外す）・`face`（`null` で外す）・`defaultScale`。**渡した項目だけ変わる。** 立ち絵の絵そのものは差し替えられない（新しい絵なら作り直す）。
  PSD から取り込んだキャラ（`kind: "psd"`）は `name`・声・`figure`・`voice`・`face`・`defaultScale` だけ（表情は画面で）。今の設定は
  `GET /api/v1/agent/characters/{characterId}` か `list_characters`（`kind`・`voice`・`emotions`・`face`・`defaultScale`）で読める。
  声（`speakerId`・`voiceOf`）を変えても、音声を作り済みのプロジェクトは前の声のまま——`generate_audio` に `force: true` を付けて作り直す
- **消す**のは画面から（AI からは消せない）。**PSD の取り込みも画面だけ**（PSD は利用者のブラウザの中で分解する）
- スコープは作る・直すのが `projects:write`、読むのが `projects:read`。レート上限は作るのが素材のアップロードと同じ枠、
  直すのが書き込みの枠（超えると 429 `rate_limited`）。**クレジット消費**: 0
- 使う画像の利用規約（動画での利用・加工ができるか）は利用者が確認すること

## 注意

- **音声はこちらで合成する。** 相手側に YMM4 や VOICEVOX を入れてもらう必要はない。
- 音声エンジンはキャラごとに決まっている（`list_characters` の `engine`）。
  ゆっくり霊夢・魔理沙は AquesTalk、ずんだもん等は VOICEVOX。
- **初回は `output: "draft"` で作り、`create_preview_link` の共有リンクで確かめること**（0クレジット）。
  本番のクレジット（動画1分につき1）を払う前に、キャラ・背景・カメラが意図どおりか確かめられる。
- **`create_yukkuri_video`（ジョブで受け付ける形）から MP4 までは、2026-09-25 に本番で最後まで通した**
  （OAuth でつないだ MCP から `create_yukkuri_video` → `get_job` の `succeeded` → `get_render` の完成まで。
  22行の台本2本）。まとめて作るバッチ（`create_yukkuri_videos_batch`）は 2026-09-26 に本番で**プレビュー出力（当時あった1クレジットの低解像度 MP4。2026-09-27 に廃止）の
  3本**を最後まで通した。**MP4 出力のバッチは本番未検証**（本番で最後まで通した実績はまだ無い）。
  どちらも、初めての形の台本は**少数の行で試し、`get_render` が
  `done: true` かつ `outputFile` が入る（完成）ことを確かめてから本番の台本を流すこと。**
  途中で止まる場合は `get_project` で台本を、`generate_audio` で音声を、
  `render_mp4` でレンダーを、と段ごとに切り分けられる。不具合として報告してほしい。
- MP4 の音量は行ごとに測って揃えている（2026-09-25 の実測: -13.6 / -15.9 LUFS、真のピーク
  -2.5 / -1.5 dBTP）。YouTube の基準（-14 前後）に近いので、そのまま投稿して音量で不利になることはない。
- 機械可読な定義: `https://app.yukkurigen.com/openapi.json`

## 変更履歴

- **2026-10-20**: 文書と API のずれを直した。`PUT /api/v1/projects/{projectId}/lines`（openapi の書き方が実装と別物だった）を
  削除——行を直すのは `PATCH .../lines`（`update_lines`）。`GET /api/v1/projects/{projectId}` が `subtitleWarnings` を、
  レンダーの進捗が完成時に `credits`（概要欄のクレジット）を REST でも返すようにした（以前は MCP だけ）。BGM 生成の 402 に
  `shortfall` と `purchase`、レート上限の 429 に `retryAfter` と `Retry-After` をそろえた。失敗した BGM 生成のジョブの
  冪等キーは空くようにした（同じ鍵で投げ直せる）。知らない表情・ポーズの名前は、作る・直す・挿入のどこでも 400 にした。
  章の見出し・強調テロップは長すぎると黙って切ることを明記した

- **2026-10-20**: AI（API キー・OAuth）から YouTube に投稿できるのを、**クレジットを使って書き出した MP4 だけ**にした
  （その代わりプランは問わない。クレジットパックだけの無料プランでも、課金して書き出したものなら投稿できる）
  （それ以外は 403 `render_not_paid`、焼いている途中は 409 `render_not_ready`）。`youtube:upload` は API キーの画面と
  OAuth の許可画面の「YouTube への投稿も許可する」で付けられるようにした（以前は文書だけが「発行し直して」と言い、
  画面に選ぶ場所が無かった）。画面からの投稿は変わらない
- **2026-10-20**: 音声生成の月の文字数の上限を、**クレジットパックの残高がある人・API連携ライセンスを持つ人は
  500,000字**（スタンダード相当）に引き上げた（以前はプランだけを見て、無料プランなら購入があっても 10,000字）。
  上限に達したときの 429 を `rate_limited` から **`tts_quota_exceeded`** に変え、`limit` / `used` / `resetAt` を返す
  （以前は全員に「毎月1日に戻る」と書いていたが、購読者は請求日に戻る）。旧レンダーパック（`render_pack_*`）の
  販売を止めた——`create_checkout` は 400 `validation_error` と `allowedPriceKeys` を返す（買い済みの分はそのまま使える）

- **2026-10-17**: `generate_bgm`（`POST /api/v1/assets/bgm/generate`）を足した。ACE-Step で BGM を1曲（30〜240 秒）作り、
  利用者の BGM 素材として登録する（`list_bgm` に出て、`url` を行の `bgm` に書ける）。いつもジョブで、`get_job` の
  `result` に永続の `url` と1時間の `downloadUrl` が入る。2クレジット（失敗したら返金）。`projectId` を渡しても
  動画の BGM は変えない（`applyToProject: true` のときだけ変える）。あわせて BGM 生成で作った WAV を `.mp3` の名前で
  置いていたのを、中身の形式どおりの名前にした

- **2026-10-16**: **声だけの話者**を書けるようにした。動画の `voiceOnly`（`create_yukkuri_video` の最上位・
  `update_project_settings`。`null` か `[]` で外す）に入れたキャラは一度も画面に出ず、舞台にも枠にも入らない
  （空いた枠はほかの話者が使う）。行の `offstage: true`（`create_yukkuri_video` / `update_lines` / `insert_lines` /
  `split_line`）はその台詞では舞台に出てこない（既に舞台に居れば残る）。声だけの行の `cameraMode:"dynamic"` は寄らない。
  4人の並び・描ける人数は画面に出る話者だけで数える。声だけのキャラを `stage` に書く・`action` や `camera` の
  `target` にするのは 400。`get_project` が `voiceOnly` と行の `offstage` を返す。古い `_narrator_hidden` は互換のため
  受け付けるが新しく書かない。どちらも書かない動画の見た目は変わらない


- **2026-10-15**: BGM と効果音を AI が決められるようにした。行の `bgm`（組み込みの曲の名前・利用者が上げた音の url・
  `"none"` で止める。次に書いた行まで引き継ぐ）を `create_yukkuri_video` / `update_lines`（`null` で外す）/ `insert_lines` /
  `split_line`（最初のパーツが引き継ぐ。それまで分割すると BGM の切り替えが消えていた）・REST で書け、`get_project` が返す。
  行の `effect` に利用者が上げた音の url も書けるようにした。手元の音を上げる `create_audio_upload` → `confirm_audio_upload` /
  `upload_audio`（mp3 / wav / ogg。BGM 30MB・効果音 5MB）と、書ける音の一覧 `list_bgm`（`GET /api/v1/audio-library`）を足した。
  外部の URL・他人の素材は 400。話している間だけ BGM を下げるダッキング（約 -10dB・0.15 秒で下げ・0.4 秒で戻す）を
  テンプレートの `bgm.ducking` と `update_project_settings` の `bgmDucking` で入れられる（既定は切）。焼く前に、もう無い
  上げた音を外すようにした（それまで消えた効果音の URL が残っているとレンダーごと落ちた）。`bgm` を書かない・ダッキングを
  入れない動画の音と見た目は変わらない

- **2026-10-15**: カメラが話者に寄るとき（行の `camera`・`cameraMode:"dynamic"`）、**顔の中心**を狙うようにした。
  以前は立ち絵の枠の上端から決まった割合の所を狙っていたので、ずんだもん・あんこもん・自作キャラでは顔より上
  （髪・頭の飾り）が真ん中に来て、顔が画面の下に寄っていた。組み込みキャラは絵から測った位置、自作キャラは
  新しい `face`（`create_character` が絵から推し量って保存。`update_character` で直せる。`list_characters` が返す）。
  自作キャラの `defaultScale`（立ち絵の大きさ。0.5〜2）が描くときに効くようにした（以前は保存されるだけだった）。
  寄らない動画（`camera` も `cameraMode:"dynamic"` も無い）の見た目は変わらない（`defaultScale` を 1 以外にしたキャラを除く）

- **2026-10-15**: 声の速さ・高さを AI が決められるようにした。行の `voice`（`{speed: 0.5〜2.0, pitch: -0.15〜+0.15}`。
  その行だけ）を `create_yukkuri_video` / `update_lines` / `insert_lines` / `split_line`（REST も同じ）で書け、読み上げ
  エンジンがその速さ・高さで合成する（変えた行は音声の作り直しが要る）。自作キャラにも既定の `voice` を持たせられる
  （`create_character` / `update_character`。行とは速さが掛け算・高さが足し算）。動画全体の再生速度 `voicePlaybackRate`
  の**既定が 1.1** であることを明記し、`create_yukkuri_video` でも受けるようにした（それまで書いても黙って捨てていた）。
  `voicePlaybackRate` はプロジェクトに残り、共有リンクとあとの `render_mp4` も同じ速さになる（それまで `render_mp4` で
  渡した速さは共有リンクに効かなかった）。焼かずに変える `update_project_settings`（`PATCH /api/v1/projects/{id}/settings`）
  を足し、`get_project` が `voicePlaybackRate` を返す。`voice` を書かない台本の声・既存の動画は変わらない。
  あわせて「立ち絵とBGM」の誤り（`POST /api/v1/projects/{id}/bgm` は無い。GET と DELETE だけ）を直した

- **2026-10-15**: 読みを直す道具を足した。`check_readings`（`POST /api/v1/projects/{id}/readings/check`）は、行ごとに
  エンジンが実際に読む音（カタカナ）を返す（消費なし・音声は作らない）。`text` に**ルビ** `｜漢字《よみ》`（短く `漢字《よみ》`）を
  書けるようにした——字幕は親文字、声はよみ。自分の**読みの辞書**（`set_pronunciation` / `list_pronunciations` /
  `delete_pronunciation`、REST は `/api/v1/pronunciations`）を足した。辞書を変えると、作り済みの音声は `get_project` の
  `staleAudio` に出て、`generate_audio`（引数なし）が作り直す。ルビも辞書も書かない台本の音声・字幕は変わらない

- **2026-10-15**: 行ごとのカメラ `camera` を足した（`create_yukkuri_video` / `update_lines` / `insert_lines` /
  `split_line`・REST）。`{zoom, target, center, move, shake, materials}` で、**寄る先（話者・キャラ・画面の点）を画面の
  真ん中に持ってきて**寄る・動き方（cut / ease / slow）・画面の揺れ・素材も一緒に寄せるかを、その行だけ決められる。
  キャラは動かさない（場面ごとカメラが動く）。`cameraMode:"dynamic"` は今までどおり（両方書いたら `camera`）。
  `camera` をどの行にも書かない動画の見た目は変わらない
- **2026-10-15**: 行の `action` で**キャラを動かせる**ようにした（`jump` 跳ぶ / `hop` ぴょんぴょん・`stage` で
  立ち位置が変わる行では跳ねながら移る / `shake` 震える / `bow` お辞儀 / `spin` 振り向く / `fall` ずっこける /
  `bounce` はしゃぐ）。立ち絵の全体を動かすので、組み込み・自作（PNG・PSD）・ゆっくりのどのキャラにも効く。
  文字列はその行の話者、`{"type","target"}` は target のキャラ、配列で複数。その行の頭で1回だけ動き、引き継がない。
  `create_yukkuri_video` / `update_lines` / `insert_lines` / `split_line`（REST も同じ）で書け、`get_project` はそのまま返す。
  `action` を書かない動画の見た目は変わらない。

- **2026-10-15**: 行の `stage` で**キャラの立ち位置・大きさ・向き**を決められるようにした。並びの要素を
  `{"id","x","y","scale","facing"}` のオブジェクトにすると、そのキャラはテンプレートの枠ではなく書いた所に立つ
  （`x` 0〜100 は横の中心の %、`y` -50〜50 は下へずらす %、`scale` 0.5〜2、`facing` は `left`/`right`）。
  位置は `stage` と一緒に次の `stage` まで引き継ぎ、変わると約0.3秒で動く。位置を書いたキャラは4人の並び・素材の寄せに
  従わない。文字列だけの `stage` と、`stage` を書かない動画の見た目は変わらない。`create_yukkuri_video` /
  `update_lines` / `insert_lines` / `split_line`（REST も同じ）で書け、`get_project` はそのまま返す。

- **2026-10-15**: `tohoku-ahiru-talk`（とその複製）に4人まで同時に出せるようにした（以前は3人。4人目は声と字幕だけだった）。
  舞台に居る人数で並びが変わり、4人そろうと左2人・右2人に少し小さく並ぶ（素材が出ている間は左右の端に2人ずつ）。
  `stage` で人数が変わると約0.3秒で並び直す。4人目の枠 `right2` を組み込みに足した（`list_templates` の `layout` に出る）。
  2人・3人の動画の見た目は変わらない。`yukkuri-talk` は2人のまま。

- **2026-10-15**: 自分のキャラを AI が作れるようにした。`list_voices`（`GET /api/v1/voices`）・
  `create_character`（`POST /api/v1/agent/characters`）・`update_character`（`PATCH /api/v1/agent/characters/{characterId}`。
  読むのは `GET`）を追加。上げた PNG・WebP の画像1枚と声（`speakerId` か `voiceOf`）から、画面で作るのと同じキャラができ、
  `characterId` をそのまま `speaker` に書ける。表情ごとの画像・立ち絵の種類も付けられる。PSD の取り込みと削除は画面だけ。
  `list_characters` の自作キャラに `kind`・`voice` を足し、`emotions` は画像・部品を設定した表情だけにした（空の表情を数えていた）。
  消費なし。

- **2026-10-14**: テンプレートエディタ（画面）を廃止した。見た目は利用者が AI に頼み、AI が直す。
  `update_template_style`（`PATCH /api/v1/agent/templates/{templateId}/style`、読むのは `GET` か `list_templates` の `style`）を追加。
  字幕の描き方（`look`）・大きさ・色・フチ・箱・文字間・行の高さ・話者名、キャラの声だけ・左右反転、カメラの寄り、BGM の音量、
  背景、名前を直せる。描かれない項目は `400 unsupported_field` で断る（黙って保存しない）。スコープは読むのが `projects:read`、
  直すのが `projects:write`。消費なし。`copy_template` と `update_template_layout` の応答から `editorUrl` を外した。

- **2026-10-14**: 行ごとの場面の演出を4つ足した（`create_yukkuri_video` の `script[]`・`update_lines`・
  `insert_lines`・`split_line`・`POST /agent/generate` ほか REST の同じ項目。`get_project` も返す）。
  `stage`（舞台に立つキャラ。途中からの登場・退場。次に書いた行まで引き継ぐ）、`backgroundMotion`（背景の
  ゆっくりした寄り・引き・パン。引き継ぐ。既定 `none`）、`transition`（`dissolve` 既定 / `fadeBlack` 暗転 / `cut`）、
  `eyecatch`（章の頭の全画面タイトル。音声の前に 1.8 秒の無音が入り、動画とクレジットの長さにも入る）。
  どれも書かなければ見た目は以前と同じ。あわせて、自作テンプレートでも行の背景の切り替えを 0.6 秒で溶かすように
  した（以前は一瞬で切り替わっていた）。

- **2026-10-14**: 手元の画像を上げる `create_image_upload`（`POST /api/v1/assets/image-uploads`）・
  `confirm_image_upload`（`POST /api/v1/assets/image-uploads/confirm`）・`upload_image`
  （`POST /api/v1/assets/image-uploads/inline`）を追加。シェルがあれば署名付き PUT（最大 10MB）、無ければ
  base64（デコード後 8MB）。返る `url` は `import_images` と同じ扱い。消費なし。スコープは `projects:write`。
  あわせて `POST /api/v1/assets/import` が API キーを受けることを openapi に載せた（実装は前から受けていた）。

- **2026-10-09**: `get_channel_video_analytics`（`GET /api/v1/channels/{channelId}/analytics/videos`）と
  `get_video_retention`（`.../analytics/videos/{videoId}/retention`）を追加。YouTube に上げた動画の実績を
  動画ごとに読み、視聴維持率の推移からどこで離脱したかを見られる。消費なし。スコープは `projects:read`。

- **2026-09-27**: `import_images`（`POST /api/v1/assets/import`）を追加。
  URL リストから画像を S3 へ取り込み、materialImageUrl / backgroundImageUrl に使える
  永続 URL を返す。SSRF 対策あり（DNS 解決 + リダイレクト再確認）。クレジット消費なし。

- **2026-09-26**: 利用者の自作キャラ（PSD 取り込み）を `list_characters` に `custom: true` で返し、
  その `id` を `speaker` に使えるようにした（一座は東北勢）。東北式の画面で枠の無い話者（春日部つむぎ・
  自作キャラ）が声だけで映らなかったのを直し、喋らないキャラの枠に座らせるようにした。

- **2026-09-15**: `create_yukkuri_videos_batch`（`POST /api/v1/agent/generate/batch`）をジョブで
  受け付けるようにした（MCP は自動で、REST は `Authorization: Bearer` に `Prefer: respond-async` を
  付けたとき。サーバの設定によってはジョブにならず同期のまま）。数秒で 202 と `jobId` / `requested` が
  返り、**この応答に `results` は入らない**——`get_job` が `succeeded` になったら `result.results` の
  各件の `renderId` と `projectId` を `get_render` に渡す。各件は1件ずつ順に作り、クレジットはその件を
  作り始めるときに押さえる。打ち切りの規則（`stoppedAtIndex` / `stopReason` / `notAttempted`）は
  同期と同じ。ジョブで受け付けるときは、受け付けの時点で本文の大きさ（合計 4 MiB）とレート上限と
  進行中のバッチの数（同じ利用者に2件まで。429 `rate_limited`）も見る。3時間待っても終わらない件は
  `child_timeout` になり次の件へ進む（その件のジョブは動き続ける）。バッチのジョブの `failed` は
  返金を意味しない（作り始めた件は課金されたまま進む）。失敗したバッチには `GET /api/v1/jobs/{jobId}` が
  `partialResults`（作り始めた件・投げ直してよい件の番号・同じ prefix で全件を投げ直せる期限）を付ける。
  同じ prefix の投げ直しで二重に作られないのは、その件が終わってから 24 時間以内だけ。サーバの設定に
  よっては、進行中のバッチが次の件の前に `jobs_paused` で打ち切られる。`results[].status` は整数で、mp4 の
  件には同期でも `renderId` が入るようにした（ジョブで受け付けたなら `jobId` も）。

- **2026-09-15**: `create_yukkuri_video`（`POST /api/v1/agent/generate`）の `output:"mp4"` /
  `"preview"` をジョブで受け付けるようにした（MCP は自動で、REST は
  `Prefer: respond-async` を付けたとき。サーバの設定によってはジョブにならず同期のまま）。数秒で 202 と `jobId` / `projectId` が返り、
  **この応答に `renderId` は入らない**——`get_job` が `succeeded` になったら `result.renderId` を
  `get_render` に渡す。クレジットは受け付けの時点で押さえ、ジョブが失敗したら返金する。
  残高不足などの門はその場で返り、そのときプロジェクトは作られない。同じ鍵の再送は同じ
  `jobId` を返す。ジョブの GET は `projectId` を返し、通知の署名鍵（`callbackSecret`）は返さない。
  `output:"draft"` は同期のまま。`Prefer` を付けない REST の呼び出しも既定では同期のまま
  （サーバの設定で待つ時間を決めてあるときだけ、終わらなければ 202 が返る）。
- **2026-09-15**: `render_mp4`（`POST /api/v1/projects/{projectId}/render`）の
  `waitForCompletion` を非推奨にし、**受け付けるが無視する**ようにした。付けても
  待たずに `status: "rendering"` と `renderId` を返す（400 にはならない）。
  完了は `get_render` / 進捗で追うこと。待機中の失敗を表していた 500 の
  `render_incomplete` は、これに伴い返らなくなった。
