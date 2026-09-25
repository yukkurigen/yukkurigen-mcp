# YukkuriGen — ゆっくり動画を作る (Agent Skill)

テーマから「ゆっくり解説/劇場」動画の **MP4** を生成するためのスキル。
あなた（AI）が台本を書き、YukkuriGen が音声を合成してレンダーし、動画のURLを返します。
ユーザーはそれをそのまま投稿できます。

## 接続

- 発行: ユーザーに `https://app.yukkurigen.com/settings/api-keys` でAPIキーを発行してもらう（コネクタから OAuth でつなぐなら不要）。
  **無料プランでもここまで使える**（無料枠10クレジット）:
  - 台本の作成・読み返し・修正（`create_yukkuri_video` のプロジェクト作成部分、
    `get_project`、`update_lines`、`generate_audio`）——**すべて消費なし**
  - **MP4 プレビュー**（`output:"preview"` または `render_mp4` + `preview`。
    1クレジット・640×360・最大20秒＝10回ぶん）
    ——まず動くものを見せたいときはこれを使う。
  - 本番の MP4 レンダー（全編・本番解像度・5クレジット）だけは有料プラン
    〈スタンダード以上〉または買い切りライセンスが必要。`402 plan_required` が返る
    ——そのときは下の「支払いが要るとき」に従って購入まで案内する。
- MCP（ChatGPT・claude.ai などのコネクタ）: `https://app.yukkurigen.com/api/mcp` を追加するだけ。
  OAuth 2.1（動的クライアント登録・PKCE）で YukkuriGen の許可画面が開き、利用者が「許可する」を押すと
  つながる。**鍵の発行は要らない。** 発行されたトークンは利用者の鍵一覧に「<アプリ名>（OAuth）」として出て、
  そこから取り消せる。スコープは画面で発行する鍵と同じ（`youtube:upload` は付かない）。
- MCP（ヘッダーを渡せるクライアント）: `claude mcp add --transport http yukkurigen https://app.yukkurigen.com/api/mcp --header "Authorization: Bearer <KEY>"`
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
| 音声を作る（MP4の前に必須） | `generate_audio` | `POST /api/v1/projects/{id}/audio/generate` |
| 非同期ジョブの状態取得 | `get_job` | `GET /api/v1/jobs/{jobId}` |
| **焼く前に人へ見せる共有リンク** | `create_preview_link` | `POST /api/v1/projects/{id}/preview-link` |
| YouTube へ投稿する | `upload_to_youtube` | `POST /api/v1/projects/{id}/upload` |
| 投稿ジョブの状態 | `get_youtube_upload` | `GET /api/v1/projects/{id}/upload/{jobId}` |

| 見た目の一覧 | `list_templates` | `GET /api/v1/agent/templates` |
| チャンネルの一覧 | `list_channels` | `GET /api/v1/channels` |
| 既存プロジェクトを MP4 に | `render_mp4` | `POST /api/v1/projects/{id}/render` |
| レンダー進捗/出力URL | `get_render` | `GET /api/v1/projects/{id}/render/{renderId}/progress` |
| 決済リンクを出す（402のとき） | `create_checkout` | `POST /api/v1/billing/checkout-link` |
| 支払いが済んだか確認 | `get_checkout` | `GET /api/v1/billing/checkout-link/{sessionId}` |

`list_projects` / `get_project` / `update_lines` / `generate_audio` /
`create_preview_link` は**消費なし**。
確認用プレビュー（`output:"preview"` / `render_mp4` + `preview`）は**1クレジット**。
課金はレンダーだけで起きる。直すこと自体にお金がかからないので、ユーザーが
納得するまで何度でも直してよい。

### 焼く前に、人に見てもらう（消費なし）

**MP4 を焼く前に共有リンクを使うこと。** ログイン不要で開けるURLが返るので、
そのまま利用者へ渡す。相手はブラウザで**音つき・動く状態**を確認できる
（動画ファイルは出ない）。

    create_yukkuri_video { output:"draft" }   # 消費なし
    generate_audio                            # 消費なし
    （共有リンクを発行して渡す）                # 消費なし
    （相手の OK を待つ。直したら update_lines → generate_audio。
      **同じURLがそのまま新しい内容を映す**ので、作り直さなくてよい）
    render_mp4                                # 5クレジット

**音声を作る前にリンクを渡さないこと**——無音で再生され、確認にならない。

`generate_audio` は**引数なしで呼べる**（音声が無い行すべてが対象）。REST を
直接叩くときも本文は空 `{}` でよい。特定の行だけ作り直したいときだけ、
`get_project` が返す行の id を `lineIds` に並べて `force: true` を付ける。

### 非同期ジョブ（Prefer: respond-async）

`generate_audio`（POST /audio/generate）、台本生成（POST /script/generate）、
BGM 生成（POST /bgm/generate）、BGM パイプライン（POST /bgm/pipeline）は、
`Prefer: respond-async` ヘッダを付けて呼ぶとサーバー側で非同期ジョブになる
（サーバの設定によっては付けてもジョブにならず、同期で処理して 200 を返す）。

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

MCP では `generate_audio` と `create_yukkuri_video`（mp4 / preview）と
`create_yukkuri_videos_batch`（下の「大量に作る」）のツールが自動的に
この非同期パスを使い、jobId を返す。`get_job` ツールで追う（サーバの設定によっては
ジョブにならず、同期で待ってから結果を返す）。

#### 台本→MP4 の一括生成もジョブで受け付ける

`create_yukkuri_video`（`POST /api/v1/agent/generate`）の `output:"mp4"` / `"preview"` も、
`Prefer: respond-async` を付けるとジョブになる（サーバの設定によっては付けてもジョブにならず、
同期で待ってから 200 と `renderId` を返す）。
`output:"draft"` は常に同期。**MCP の `create_yukkuri_video` は自分でこれを付ける**ので、
MCP から使うときは何もしなくてよい。

受け付けでは、同期と同じ門（認証・スコープ・プラン・レート上限・`callbackUrl`・
クレジット）をその場で通す。残高不足・プラン不足・レート上限などは**数秒でその場に返り**、
そのときプロジェクトもジョブも作られない。通ったらクレジットを押さえ、プロジェクトと
台本を作ってから 202 を返す:

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
  `plan_required` など）、押さえたクレジットは**返金される**（反映まで少しかかることがある）。
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

焼き上がったら `upload_to_youtube` で投稿できる。**ただし条件が3つある**:

1. **APIキーに `youtube:upload` スコープが要る。** 既定では付いていない
   ——403 が返ったら、利用者に付きのキーを発行し直してもらう
2. プロジェクトに**チャンネルが紐づいている**こと（未分類は不可）
3. そのチャンネルで **YouTube 連携が済んでいる**こと（未連携なら 400 で理由が返る）

**公開範囲の既定は `private`**（本人しか見られない下書き）。利用者が公開を
望んでいると確認できたときだけ `public` にすること。投稿は非同期なので、
返った `jobId` を `get_youtube_upload` で追う。`completed` になれば
`videoId` が入る。

リンクは既定7日で切れる（最大30日）。渡す相手を間違えたら `revoke: true` を
付けて作り直す。**以前配ったURLは全部開けなくなる。**

なお、プレビューはブラウザで組み立てて再生している。書き出す側とは
プログラムの配布経路が違うので、**細部は完全に同一ではない**。最終的な
見た目は書き出した MP4 で確かめること。

### 応答が返らないときのために（重要）

`create_yukkuri_video` の `mp4` / `preview` を**同期で**呼ぶと、**音声合成とレンダー開始を
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
      render_mp4                                # ここで初めて5クレジット
      get_render                                # 完了までポーリング

  1回あたりが短くなるので切れない。`draft` は `projects:write` だけで通る
  （`render:mp4` は要らない）。

### 縦型で作りたいとき

`create_yukkuri_video` の `platform` に `shorts`（または `tiktok`）を渡す。
既定は `youtube`（横型）。`render_mp4` にも同じ引数がある。

### 見た目の既定はチャンネルに置ける

文字サイズ・素材フィルター・エンディングカードは、チャンネルの設定画面
（`/channels/{id}` の「設定」タブ）で既定を決められる。1本だけ変えたいときは
`render_mp4` の引数が優先される（**リクエスト > プロジェクト > チャンネル**）。

## 手順

1. **キャラを選ぶ**: `list_characters` で `id` と音声エンジンを確認。
   **1本の動画に出せるのは同じ一座のキャラだけ**（立ち絵の系統が違うと画面に出せない）:
   - ゆっくり（東方）: `reimu`（霊夢）, `marisa`（魔理沙）
   - 東北勢: `zundamon`（ずんだもん）, `metan`（四国めたん）, `tsumugi`（春日部つむぎ）, `anko`（あんこもん）

   混ぜると `400 cast_mismatch` を返す（課金なし）。応答の `casts` に一座の一覧が入っている。

   **見た目（テンプレート）は人が決める。あなたは選ぶだけ。**
   `list_templates` で一覧を取り、返った `id` を `templateId` に渡す。
   `source: "system"`（5種）と `source: "mine"`（その人がエディタで作ったもの）の
   **どちらの id もそのまま渡してよい**。利用者が「いつもの見た目で」と言ったら
   `source: "mine"` から選ぶこと。**新しく作ろうとしないこと。**
   省略すれば台本の話者から自動で選ぶので、指定は必須ではない。

   存在しない id や他人のテンプレートを渡すと `400 template_not_found` を返す
   （課金なし）。**黙って別の見た目で焼くことはしない**——2026-09-09 まではそう
   なっていて、自作テンプレートを指定しても `line-scroll`（スポーツ反応集）で
   焼かれていた。

   チャンネル（テーマ・想定視聴者・既定の指示）を使うなら `list_channels` で
   `id` を取り、`channelId` に渡す。

   **返るのは MP4。** こちらで音声を合成してから本番レンダーを開始し、
   `renderId` を返す（5クレジット・有料プラン）。**`output:"preview"` なら
   1クレジット・低解像度・20秒**で、無料プランでも動くものが見られる。
   ジョブで受け付けたとき（MCP など）は先に `jobId` が返り、`get_job` の
   `result.renderId` で受け取る（上の「非同期ジョブ」）。
   どちらも最後は `get_render` でポーリングする。
2. **台本を書く**: 話者(`speaker`=キャラid)とセリフ(`text`)の配列を作る。掛け合い形式が「ゆっくり」らしい。
   - 漢字の読み間違いを避けたい箇所は `reading`（発音かな）を付ける。
   - 冒頭の挨拶 → 本題（結論→理由→具体例）→ まとめ、の構成が定番。
   - **独自の意見・体験を1つ以上入れる**と、量産テンプレに埋もれない動画になる。
3. **背景を決める**: `backgroundImageUrl`（https のURL）を渡す。
   省略すると既定の背景（室内のイラスト）になる。話題に合う背景を渡すと見栄えが上がる。
   場面転換を付けたいときは、その行の `backgroundImageUrl` に別のURLを入れる（以降の行へ引き継がれる）。
   BGM は指定しなくても既定の曲が流れる。
   `title` は横長の動画の左上にテーマとして出し続ける（途中から見た人にも何の話か分かる）。
   消したいときは `titleTelop: false`。
4. **カメラを置く**: 盛り上がり・感情のピーク・オチの行に `cameraMode: "dynamic"` を
   付けると、**背景ごとカメラがその話者に寄る**。立ち絵を消したい行（場面の要約など）は
   `"summary"`。
   **2〜4回に留めること。全行に付けると寄りっぱなしになり、寄りが効かなくなる。**
   立ち絵が画面に出ている行にだけ効く。
   **表情と効果音**: `emotion`（`joy` / `surprise` / `sad` / `angry` / `explaining` /
   `love`）で立ち絵の表情が変わり、`effect`（例: `"ショック1"`, `"きらーん1"`,
   `"ひらめく1"`, `"チーン1"`）で効果音が鳴る。**`emotion`・`effect`・`cameraMode` は、
   1行も書かなければセリフから自動で要所に付く。** 1行でも書いた項目は、書いた指定だけが
   使われる（自動と混ぜない）。演出を自分で決めたいときだけ書けばよい。
5. **生成する**: `create_yukkuri_video`（または `POST /agent/generate`）に `{ title, backgroundImageUrl, script }` を渡す。
   既定で MP4 を焼く。試すだけなら `output: "preview"` を足す（1クレジット・20秒）。
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
2. **直す**: `update_lines` に**変える行だけ**を渡す。渡さなかった行は一切変わらない
   ので、ユーザーが気に入っている部分を壊す心配がない。
   ```json
   { "edits": [ { "index": 3, "text": "実は理由はもっと単純なのだ" },
                { "index": 5, "backgroundImageUrl": "https://example.com/bg2.jpg" } ] }
   ```
3. **音声を作り直す**: `text` / `speaker` / `reading` を変えた行は、**その行の音声が
   無効化される**（応答の `audioInvalidated` に行番号が入る）。古い音声を残すと
   新しい字幕と違うことを喋る動画になるため、こちらで必ず消している。
   `generate_audio` を呼んで作り直す。
4. **安く確かめる**: `render_mp4` に `preview: { fromLine, toLine }` を付けると、
   **1クレジット**・低解像度で**その範囲だけ**焼ける。直した箇所を見るのはこちら。
5. **出す**: `preview` を付けずに `render_mp4`（5クレジット・全編・本番解像度）。

台本を作り直す（`create_yukkuri_video` をもう一度呼ぶ）のは最後の手段。
ユーザーが良いと言った行まで消えるうえ、レンダーにもう一度課金される。

## MP4 は音声を自動生成しない

`render_mp4` は**行に載っている音声を再生するだけ**で、音声を作りはしない。
音声が無い行があるまま呼ぶと、**課金せずに `409 missing_audio`** を返して止まる
（無音の動画に5クレジット払わせないため）。応答の `missingAudioLines` に対象行が
入っているので、`generate_audio` を呼んでから再実行する。

`create_yukkuri_video` は**中で音声まで作る**ので、この手順は要らない。
`get_project` → `update_lines` で直したあと `render_mp4` を呼ぶときだけ、
自分で `generate_audio` を挟む必要がある。

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

BGM は既定のものが入る。差し替えたい場合はエディタで設定する
（`POST /api/v1/projects/{id}/bgm`）。

## クレジットとエラー

- 消費: MP4 レンダー = 5、プレビュー = 1、AI台本生成 = 1。台本の作成・修正・音声生成は 0。
- 不足時: `code: "insufficient_credits"` / `code: "plan_required"`。**諦めずに購入まで案内すること**（下の「支払いが要るとき」）。
- 権限不足: `code: "insufficient_scope"`。キーのスコープを確認。
- 認証まわり: `code: "unauthenticated"`（401、キーが無効/失効）、
  `code: "forbidden"`（403、他人のプロジェクトなど対象への権限が無い）、
  `code: "not_found"`（404、projectId / renderId が無い）。
  いずれもクレジットは消費されない。
- 台本を作るとき（`create_yukkuri_video`）: `code: "cast_mismatch"`（400、一座をまたぐ話者。
  声は出るが画面に映らない動画になるため、**課金せずに**中断する。`undrawableSpeakers` と
  `casts` を見て、どちらかの一座に揃える）。
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

`plan_required`（無料プランで MP4）のときは、パックではなく**プラン**が返る。
無料プランは残高があっても MP4 を出せないので、パックを買わせても解決しない。

`get_checkout` は支払いの有無を直接返す。**残高の増減から推測しないこと**——
残高は月次リセットや他の操作でも動く。
  - `ai_unavailable` … AI サービスが一時的に使えない（503）。台本生成ジョブが失敗する。
    クレジットは返金済みか、後追いの掃除処理が返金する。少し置いてから再試行してよい。
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
- キャラid の打ち間違い: `code: "validation_error"` と **`validSpeakers`（そのリクエストで使えるid一覧）** が返る。
  未知のidは課金前に400になる（黙って別キャラや立ち絵なしで出力してクレジットだけ
  消費する、ということはしない）。照合される集合はリクエストによって違う:
  - `POST /agent/generate`: `speaker` はシステムキャラ（`GET /api/v1/characters`）。
  - `PATCH /projects/{id}/lines`: **そのプロジェクトの行が使っている character_id**
    ＋ 自分で作ったカスタムキャラ。
  返ってきた `validSpeakers` をそのまま使えばよい（システムキャラ一覧だけを見て
  判断すると、カスタムキャラを誤って除外する）。
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
本文に入れても効く。レンダーは5クレジットと一番高いので、特に付けること。同じ鍵で
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

- mp4 / preview の件は `projectId` と `renderId`（ジョブで受け付けたなら、その件のジョブの
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
  要るスコープも持っていない場合（403 `insufficient_scope`。mp4 / preview は `render:mp4` と
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

- `platform` / `outputFormat`: 縦横と用途（youtube / tiktok / shorts）。
- `fontSize` / `titleFontSize`: 字幕とタイトルの文字サイズ。
- `voicePlaybackRate`: 読み上げ速度（0.5〜2.0）。
- `bgmVolume`: BGM 音量（0〜1）。
- `materialImageFilter`: 素材画像のフィルタ（`blur` / `top-dark`）。
- `showOutroCard`: 終了カードの有無。

MCP のツール定義と openapi は同じ集合を公開している。片方にしか無い
項目があれば、それは不具合として扱ってよい。

## 注意

- **音声はこちらで合成する。** 相手側に YMM4 や VOICEVOX を入れてもらう必要はない。
- 音声エンジンはキャラごとに決まっている（`list_characters` の `voiceEngine`）。
  ゆっくり霊夢・魔理沙は AquesTalk、ずんだもん等は VOICEVOX。
- **初回は `output: "preview"` で試すこと**（1クレジット・20秒）。本番の
  5クレジットを払う前に、キャラ・背景・カメラが意図どおりか確かめられる。
- **`create_yukkuri_video` から MP4 まで一息に通す経路は本番未検証**（2026-09-07 時点）。
  台本の作成・音声の合成・レンダーの各段は個別に動いているが、1コールで
  最後まで通した実績がまだ無い。**少数の行で試し、`get_render` が
  `completed` を返すことを確かめてから本番の台本を流すこと。**
  途中で止まる場合は `get_project` で台本を、`generate_audio` で音声を、
  `render_mp4` でレンダーを、と段ごとに切り分けられる。不具合として報告してほしい。
  ジョブで受け付ける形（2026-09-15）も、本番で最後まで通した実績はまだ無い（まとめて作るバッチも同じ）。
- MP4 の音量は行ごとに測って揃えている（実測 -14.4 LUFS、真のピーク
  -2.0 dBTP）。YouTube の基準（-14 前後）とほぼ同じなので、そのまま
  投稿して音量で不利になることはない。
- 機械可読な定義: `https://app.yukkurigen.com/openapi.json`

## 変更履歴

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
  よっては、進行中のバッチが次の件の前に `jobs_paused` で打ち切られる。`results[].status` は整数で、mp4 / preview の
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
