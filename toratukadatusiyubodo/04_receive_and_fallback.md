# 04 もちの受け取り手順と、書けなかったときの退避

（A）では、ダッシュボードに書くのは**もち1人**です。朝の2つのジョブは値を「値ファイル」と「報告の末尾のブロック」に残すだけで、ダッシュボードには触りません。この文書は、もちがその値をどう反映するか、うまくいかなかったときに値をどう失わずに残すかを決めます。

---

## 1. 全体の流れ（平易な説明）

1. 朝のGmail要約（8:12）は、Claude・Perplexity・VMの値を測り、**値ファイル**（小さなJSON）を書いて、要約の最後に【トラッカー用の値】を載せる。
2. GitHubチェック（9:12）は、結果を**値ファイル**に書く。チャットへの報告は、従来どおり変化があった日だけ。
3. もちは、定時ジョブの報告を受け取ったときなどに、値ファイルの置き場（下の3か所）を見て、まだ反映していないファイルをダッシュボードに反映する。
4. 反映できたファイルは「反映済み」の置き場へ移す。**反映できなかったファイルはそのまま残る**ので、それがそのまま「退避」になる。測り直しは要らず、次の機会にそのまま反映できる。
5. 週次棚卸し（日曜）が、置き場に古いファイルが残っていないかを点検し、残っていれば1行で知らせる。

「退避ファイル」を別に作る仕組みを足さず、**値ファイル自体を退避として使う**のがこの設計の要点です。

---

## 2. 失敗しうる点と、その対策

| 失敗しうる点 | 起きること | 対策 |
|---|---|---|
| もちが報告を受け取らない・処理しない | 値ファイルが「未反映」の置き場に残る | 次にもちが処理するとき、置き場にあるファイルを古い順にすべて反映する。週次棚卸しが残りを点検して1行で知らせる |
| GitHubが変化なしの日（報告が来ない） | もちにきっかけが来ない | もちは、どのジョブの報告で動くときも**すべての置き場**を見る。翌朝のGmail要約の受け取りで拾われる（最大1日遅れ。画面には「1日前の情報」と出る） |
| もちの部分更新が失敗する（2026-11-20 のような権限の拒否を含む） | ダッシュボードは古いまま | ファイルを未反映の置き場に残したまま、1行だけ報告し、次の機会に同じファイルを反映し直す（測り直し不要） |
| ジョブが値ファイルを書けなかった | ブロックの最後の行が「値ファイル：書き込み失敗」 | もちが、ブロックの値を同じ形のJSONに写して未反映の置き場に保存してから、通常どおり反映する |
| 古い値ファイルが新しい値を上書きしそう | 部分更新の操作が OLDER_THAN_CURRENT で拒否する | 「もう新しい値が入っている」という意味なので、そのファイルは「反映不要（superseded）」として反映済みの置き場へ移す |
| ダッシュボードのデータが壊れている | 部分更新が LATEST_INVALID で拒否される | 部分更新を止め、1行報告。復旧手順（6章の「データが壊れたとき」）に従う |

---

## 3. 置き場と形式

### 置き場（フォルダ）

| 出どころ | 未反映（ここに残っている＝まだ反映していない） | 反映済み |
|---|---|---|
| 朝のGmail要約 | `~/workspace/goals/daily-gmail-summary/hidden_files/tracker-pending/` | `~/workspace/goals/daily-gmail-summary/hidden_files/tracker-applied/` |
| GitHubチェック | `~/workspace/goals/github/hidden_files/tracker-pending/` | `~/workspace/goals/github/hidden_files/tracker-applied/` |
| 週次棚卸し（Spaceに書けなかったときだけ） | `~/workspace/goals/goal-6/hidden_files/tracker-pending/` | `~/workspace/goals/goal-6/hidden_files/tracker-applied/` |

- ファイル名は実行日（日本時間）の `YYYY-MM-DD.json`。同じ日にもう一度実行されたら上書きされる。
- 置き場は、それぞれのジョブの持ち主のゴールの `hidden_files` の下にした。ジョブが自分のゴールのフォルダに書くのが一番安全なため（他のゴールのフォルダへの書き込みが常に許されるかは確かめていない）。
- 2026-11-22時点で、`daily-gmail-summary/hidden_files/` と `goal-6/hidden_files/` は空、`github/hidden_files/` には状態ファイルなど6件がある。`tracker-pending/` と `tracker-applied/` はまだ無い（ジョブまたはもちが `mkdir -p` で作る）。

### 形式（JSON）

**朝のGmail要約**（`01_gmail_job_revision.md` の本文と同じ形）
```json
{"source": "daily-gmail-summary", "formatVersion": 1, "runDate": "2026-11-23", "capturedAt": "2026-11-23T08:13:40+09:00",
 "claude": {"state": "ok", "weeklyPercent": 10, "resetAt": "2026-11-28T01:00:00+09:00", "resetLabel": "11/28（土）01:00"},
 "perplexity": {"state": "ok", "pro": 200, "research": 20, "labs": 25},
 "vm": {"load": {"state": "ok", "m1": "0.30", "m5": "0.31", "m15": "0.27"},
        "memory": {"state": "ok", "totalGiB": "7.7", "availableMi": 629, "usedPercent": 92},
        "disk": {"state": "ok", "usedG": "6.5", "totalG": "150", "usedPercent": 4, "workspace": "7.8G"}}}
```
- `state` は `"ok"`／`"failed"`／`"session_expired"`（Perplexityのみ）／`"stopped"`（Claudeのみ、11/26以降）。`"ok"` 以外は `state` だけが入る。
- `claude` が無いファイル（01 の本文B を適用した後）は、「Claude は対象外」とみなして何もしない。

**GitHubチェック**（`02_github_job_revision.md` の本文と同じ形）
```json
{"source": "github-daily-check", "formatVersion": 1, "runDate": "2026-11-23", "checkedAt": "2026-11-23T09:13:05+09:00",
 "result": "unchanged", "changedRepos": [], "note": "GitHub（2026-11-23 9:12確認）：変化なし"}
```

**週次棚卸し**（`05_weekly_audit_addition.md` の本文と同じ形。Spaceに書けなかったときだけ作られる）
```json
{"source": "weekly-house-audit", "formatVersion": 1, "runDate": "2026-11-29", "asOf": "2026-11-29T10:40:00+09:00",
 "upcomingTasks": [{"date": "2026-11-30", "dateLabel": "11/30（月）", "time": "", "title": "UQ 自宅セット割の申込期限", "note": "My UQ mobileから申込"}],
 "ongoingItems": ["…"],
 "recurringJobs": [{"name": "毎日のGmailサマリー", "schedule": "毎日 8:12"}]}
```
（上の予定・ジョブの中身は形の例。実際の中身は週次棚卸しが正本から作る）

### 反映済みの印
- 反映が終わったら、もちはファイルに `"applied"` を書き足してから、同じファイル名で `tracker-applied/` へ移す。
```json
"applied": {"at": "2026-11-23T08:20:05+09:00", "result": "applied", "detail": "Claude・Perplexity・VM を反映"}
```
- `result` は次のどれか：`"applied"`（反映した）、`"superseded"`（もっと新しい値がすでに入っていたので反映不要）、`"skipped"`（反映する項目がなかった。例：すべて取得失敗）。一部の項目だけ反映できた場合も `"applied"` にして、`detail` に反映した項目と、しなかった項目を書く。
- **未反映の置き場にファイルが残っている＝まだ反映していない**、という意味になる。印は「置き場所」そのもので、ファイルを開かなくても一覧だけで分かる。
- 反映済みのファイルは消さない（1日1KB程度で、1年でも1MBに届かない）。消すかどうかは本人に聞いてから。

---

## 4. ジョブ本文側にも値をファイルに残させるか（検討の結果）

- **採る**。01・02 の本文に入れた。理由：
  1. 報告（チャット）だけだと、もちが処理しなかった日の値が会話の流れに埋もれ、後から機械的に拾えない。
  2. GitHubチェックは変化なしの日に報告を送らない（本人の指示）ので、報告だけでは値が届かない日がある。
  3. ファイルが残っていれば、そのまま退避になる。ジョブ側に「失敗したら退避ファイルを書く」という別の手順を足さずに済み、むしろ手順が単純になる。
- 「値を報告に含めるだけ」からの追加は、**ジョブごとに1日1回、1KB程度のファイルを1つ書く**ことだけ。ダッシュボードには一切触れないので、（A）の「書き手はもち1人」は崩れない。

### 本文への追記文案（再掲）
朝のGmail要約（01 の本文A・B の最後の節。全文は 01 にある）：
```text
値ファイルと【トラッカー用の値】ブロックの書式（2026-11-23追加）：
- 値ファイル：~/workspace/goals/daily-gmail-summary/hidden_files/tracker-pending/{今日の日付（JST）YYYY-MM-DD}.json に、次の形のJSONを書く。フォルダがなければ mkdir -p で作る。同じ日付のファイルがあれば上書きする。capturedAt は値ファイルを書く時点の現在時刻（JST、ISO形式・+09:00付き）。下の数値は書式の例であり、実際には今回取得した値を入れる。
  （JSONの例とブロックの書式は 01 の本文のとおり）
- 値ファイルの書き込みに失敗した場合は、ブロックの最後の行を「値ファイル：書き込み失敗」とする（ブロックの他の行はそのまま載せる）。
```
GitHubチェック（02 の本文の該当部分。全文は 02 にある）：
```text
- チェックが正常に完了したら（手順1で基準作りだけをした回と、手順7の失敗時は除く）、変化の有無にかかわらず毎回、手順6のあとに、値ファイル ~/workspace/goals/github/hidden_files/tracker-pending/{今日の日付（JST）YYYY-MM-DD}.json を次の形で書く。フォルダがなければ mkdir -p で作る。同じ日付のファイルがあれば上書きする。
- 値ファイルの書き込みに失敗した場合：変化があった日は、報告の【トラッカー用の値】の2行の後に「値ファイル：書き込み失敗」と1行足す。変化がなかった日は、「GitHubチェックは変化なし。トラッカー用の値ファイルを書けませんでした」と1行だけ報告する（失敗の知らせとして、この場合に限り変化なしでも1行送る）。エラーの詳細は載せない。
```

---

## 5. もちの受け取り手順を置く場所（推奨）

| 置き場の候補 | 評価 |
|---|---|
| `~/MEMORY.md`（毎回読む「ホット層」） | 手順の全文を置くには長すぎる（棚卸しで約12,000字の上限を見張っている）。**きっかけの1文とポインタだけ**を置くのに最適 |
| `~/AGENTS.md`（仕事の進め方の教訓） | 「どう動くか」の置き場としては合うが、こちらも約8,000字の上限を見張っている。手順全文は置かない |
| **`~/workspace/goals/goal-8/references/トラッカー反映手順.md`（推奨）** | goal-8 はこのダッシュボードのゴール（`~/workspace/goals/goal-8/GOAL.md` 1行目「# トラッカーダッシュボード」）。references は今は空。点検報告の横断まとめ8「Spaceごとに『更新のしかた』を1枚（そのゴールのフォルダに）置く」にも合う |

**推奨**：手順の全文は goal-8 の `references/トラッカー反映手順.md` に置き、MEMORY.md 61行目（トラッカーの恒久対応の行）にきっかけとポインタを1文足す。もち自身が書き込む（この担当は書き込まない）。

### MEMORY.md 61行目の差し替え案（もちがそのまま使える文）
現在の61行目：
> - トラッカーダッシュボード（Space、slug：toratukadatusiyubodo）：日次スナップショット運用。恒久対応は2026-11-22に（A）で決定：朝の2ジョブ（Gmail要約・GitHubチェック）からダッシュボード更新を外し、もちが報告の受け取り時に部分更新で反映する。Spaceへの自動書込み経路はない（日次更新が上限。再提案しない）。

差し替え後：
```text
- トラッカーダッシュボード（Space、slug：toratukadatusiyubodo）：日次スナップショット運用。恒久対応は2026-11-22に（A）で決定：朝の2ジョブ（Gmail要約・GitHubチェック）からダッシュボード更新を外し、もちが報告の受け取り時に部分更新で反映する。Spaceへの自動書込み経路はない（日次更新が上限。再提案しない）。定時ジョブの報告を受け取ったら、tracker-pending（daily-gmail-summary・github・goal-6 の hidden_files 配下）に未反映の値ファイルがないか見て、~/workspace/goals/goal-8/references/トラッカー反映手順.md に従って反映する。
```

---

## 6. もちの受け取り手順（`トラッカー反映手順.md` にそのまま貼る本文）

```text
# トラッカー反映手順（トラッカーダッシュボード、slug: toratukadatusiyubodo）

書き手はもち1人（2026-11-22に本人が恒久対応（A）に決定）。定時ジョブはダッシュボードに書かず、値ファイルを残す。もちがこの手順で反映する。更新回数を増やす提案はしない（日次更新が上限）。

## いつやるか
- 定時ジョブの報告をチャットで受け取ったとき（朝のGmail要約、GitHubチェック、週次棚卸しのどれでも）。報告に【トラッカー用の値】が無くても、置き場の確認はする（GitHubは変化なしの日に報告が来ないため）。
- 本人にトラッカーのことを聞かれたとき。

## 手順
1. 次の3つの未反映の置き場を見て、*.json を全部挙げる。
   - ~/workspace/goals/daily-gmail-summary/hidden_files/tracker-pending/
   - ~/workspace/goals/github/hidden_files/tracker-pending/
   - ~/workspace/goals/goal-6/hidden_files/tracker-pending/
   1つも無ければ終わり。
2. 報告のブロックに「値ファイル：書き込み失敗」とあった場合は、そのブロックの値を、同じ形のJSON（下の「形」）に写して、その出どころの tracker-pending に {runDate}.json として保存してから進む。ブロックに無い値は書かない（推測しない）。
3. ファイルを、出どころごとに runDate の古い順に処理する（古い順に入れれば、最後に最新の値が残る）。1ファイルごとに 4〜6 を行う。
4. 反映する（下の「反映のしかた」）。トラッカーダッシュボードの部分更新の操作（updateService・updateMachine・updateGithubNote・replaceUpcomingTasks・replaceOngoingItems・replaceRecurringJobs）を artifact.invoke_action で呼ぶ。部分更新の操作がまだ無い間は、下の「暫定のやり方」で行う。
5. 結果に応じて印を付ける。ファイルに "applied": {"at": 今の時刻（JST、+09:00付き）, "result": ..., "detail": ...} を書き足し、同じファイル名で同じ出どころの tracker-applied/ フォルダへ移す（フォルダが無ければ作る）。
   - 全部または一部を反映できた → result "applied"。detail に反映した項目と、取得失敗などで反映しなかった項目を書く。
   - 操作が OLDER_THAN_CURRENT で断られた項目しかない → result "superseded"（もっと新しい値が入っている）。
   - 反映する項目が無かった（すべて取得失敗など） → result "skipped"。
   - 操作が失敗した（権限の拒否、NOT_FOUND、通信エラーなど） → 印を付けず、ファイルは tracker-pending に残す。別の項目の反映が済んでいても、そのファイルは残し、次の機会に同じファイルをもう一度処理する（済んだ項目は同じ時刻の上書きになるだけで害はない）。
   - 操作が LATEST_INVALID で断られた → 反映をすべて止め、「データが壊れたとき」へ。
6. 報告：うまくいったときは本人への報告は要らない（朝の要約に返事をする流れで触れるなら1行まで）。失敗したときだけ1行で知らせる。例「トラッカー反映できず（権限の拒否）。値は tracker-pending に保管済み、次の機会に反映します」。エラーの詳細や認証情報は書かない。

## 反映のしかた
朝のGmail要約（source が daily-gmail-summary）：
- claude.state が "ok"：updateService、name "Claude Pro"、patch は
  lines ["Pro契約中・次回更新 2026-11-26", "週間枠：{weeklyPercent}%使用（リセット {resetLabel}）"]、meterLabel "週間枠の使用"、meterPercent {weeklyPercent}、usageWindows [{"label": "週間枠", "percent": {weeklyPercent}, "resetAt": "{resetAt}", "periodHours": 168}]、checkedAt {capturedAt}。
  status・statusLabel・name は変えない。5時間枠は書かない。
- claude.state が "stopped"：ダッシュボードの Claude Pro の statusLabel が「解約済み（11/26）」でなければ、下の「Claude Pro 解約の1回だけの更新」を行う。すでに「解約済み（11/26）」なら何もしない。
- claude.state が "failed"、または claude が無い：何もしない（画面の「◯日前の情報」で古さが見える）。
- perplexity.state が "ok"：updateService、name "Perplexity Pro"、patch は
  lines ["Pro検索：残り{pro}回", "ディープリサーチ（Research）：残り{research}回", "Labs：残り{labs}回", "Pro契約中（SoftBankキャンペーン無料期間、〜2027-02-15ごろ）"]、checkedAt {capturedAt}。
  4行目は変えない。meterLabel・meterPercent（null）・status・statusLabel・usageWindows（空）は変えない。
- perplexity.state が "session_expired" または "failed"：何もしない。
- vm：load・memory・disk のうち state が "ok" のものだけを、updateMachine（name "Muse の VM"）の metricUpdates に入れ、checkedAt を {capturedAt} にする。"ok" が1つも無いときは updateMachine を呼ばない。取得失敗の計測値は前回の値のまま残す（2026-11-17からの決まり。推測で埋めない）。
  - 負荷平均：{"label": "負荷平均", "valueText": "{m1} / {m5} / {m15}", "percent": null, "detail": "1分 / 5分 / 15分平均"}
  - メモリ：{"label": "メモリ", "valueText": "利用可能 {availableMi}Mi", "percent": {usedPercent}, "detail": "合計 {totalGiB}GiB・利用可能 {availableMi}Mi"}
  - ディスク：{"label": "ディスク（ホーム）", "valueText": "{usedG}G / {totalG}G", "percent": {usedPercent}, "detail": "{totalG}G 中 {usedG}G 使用・ワークスペース {workspace}"}（workspace が null なら「ワークスペース 取得失敗」と書く）
  name・specs・status・statusLabel・note は変えない。
GitHubチェック（source が github-daily-check）：
- updateGithubNote、text {note}、asOf {checkedAt}。note は値ファイルのとおりに写し、言い換えない。
週次棚卸し（source が weekly-house-audit）：
- replaceUpcomingTasks（tasks {upcomingTasks}、asOf {asOf}）、replaceOngoingItems（items {ongoingItems}、asOf {asOf}）、replaceRecurringJobs（jobs {recurringJobs}、asOf {asOf}）。中身は値ファイルのとおりに写す。ただし反映する日の日本時間で日付の過ぎた予定（date が今日より前）は、tasks から外して入れる。

## 暫定のやり方（部分更新の操作ができるまでの間だけ）
1. getSnapshot を呼び、返ってきた snapshot をそのまま「元の控え」として手元に持つ。
2. 処理するファイルを全部まとめて、控えの写しのうち、上の「反映のしかた」で決めた項目だけを書き換える。並び順・他の欄・他の項目の文字は1文字も変えない（予定の並び順と note は特に注意。2026-11-17・11-20・11-22 に崩れた）。capturedAt だけは今の時刻（JST、+09:00付き）にする。
3. saveSnapshot でまとめて1回だけ保存する（ファイルごとに何回も保存しない）。
4. もう一度 getSnapshot を呼び、元の控えと比べる。書き換えた項目と capturedAt 以外に違いがあれば、すぐに「元の控え＋書き換えるはずだった項目」だけの正しい全体で保存し直し、本人に1行で知らせる。
5. 印の付け方と報告は、上の手順5・6と同じ。

## Claude Pro 解約の1回だけの更新（2026-11-26以降に1回）
updateService、name "Claude Pro"、patch は次のとおり。
- status: "unknown"
- statusLabel: "解約済み（11/26）"
- lines: ["2026-11-26 に解約（Google Play）", "使用率の取得は停止しました"]
- meterPercent: null
- meterLabel: ""
- usageWindows: []
- cadence: "manual"
- checkedAt: 反映する時刻（JST、ISO形式・+09:00付き。例 "2026-11-26T08:20:00+09:00"）
暫定のやり方で行う場合は、cadence を書かない（まだその項目が無いため）。

## データが壊れたとき（部分更新が LATEST_INVALID で断られた、または getSnapshot の status が "invalid"）
1. 部分更新を止め、本人に1行で知らせる。例「トラッカーのデータが読めない状態です。値は tracker-pending に保管済み。直してから反映します」。
2. getLastValidSnapshot（読み取り専用）で、最後に正しく読めたスナップショットを取り出す。
3. その中身に、tracker-pending と tracker-applied にある値ファイルのうち、その保存時刻より後のものを「反映のしかた」どおりに当てはめた全体を作り、saveSnapshot で1回保存する。
4. getSnapshot の status が "ok" に戻ったことを確かめてから、通常の手順に戻る。
5. sqlite などで app.db を直接開くことはしない（Space の決まり）。

## 形（値ファイル）
- daily-gmail-summary：{"source", "formatVersion": 1, "runDate", "capturedAt", "claude"（無い場合あり）, "perplexity", "vm": {"load", "memory", "disk"}}。state は "ok" / "failed" / "session_expired" / "stopped"。
- github-daily-check：{"source", "formatVersion": 1, "runDate", "checkedAt", "result", "changedRepos", "note"}。
- weekly-house-audit：{"source", "formatVersion": 1, "runDate", "asOf", "upcomingTasks", "ongoingItems", "recurringJobs"}。
```

---

## 7. Claude Pro を「解約済み（11/26）」にする1回だけの部分更新（値の一覧）

| 項目 | 値 | 理由 |
|---|---|---|
| name | `Claude Pro`（変えない。対象の指定に使う） | 今のカード名（DB写しの最新行 id 31 で確認） |
| status | `"unknown"` | 選べるのは ok／warn／unknown の3つ（`actions.ts` 66行目）。ok は緑の点で「使える」に見え、warn は黄色で「対応が要る」に見える。unknown は灰色で「止まっている」に一番近い |
| statusLabel | `"解約済み（11/26）"` | 本人の決めた表示 |
| lines | `["2026-11-26 に解約（Google Play）", "使用率の取得は停止しました"]` | 今の「Pro契約中・次回更新 2026-11-26」「週間枠：…」を置き換える。「週間枠：」で始まる行は書かない |
| meterPercent | `null` | 使用率のバーを出さない |
| meterLabel | `""` | 同上 |
| usageWindows | `[]` | 週間枠のバーを出さない |
| cadence | `"manual"` | 以後は毎日更新しないので、古さで警告色にしない（03 の改修後のみ。暫定のやり方では書かない） |
| checkedAt | 反映する時刻（例 `"2026-11-26T08:20:00+09:00"`） | 「この状態を確認した時刻」 |

- **いつ**：2026-11-26 以降の最初の機会。01 の本文A が入っていれば、11/26 朝の値ファイルの `claude.state` が `"stopped"` になるので、そのときに行う（手順書の「反映のしかた」に組み込み済み）。11/25 以前には行わない（まだ契約中）。
- **注意（未確定）**：MEMORY.md 72行目と課金メモ49行目に「もちの有料化か Claude 継続かの2択（未決定）」がある。Claude 継続になった場合は、この更新は行わない。
- カードをいつ取り除くか（removeService）は本人に任せる（未確定）。12月下旬の AI編成の見直し（ChatGPT Plus 開始）のときに、新しいカードの追加（updateService の allowCreate）とあわせて相談するのがよい。
