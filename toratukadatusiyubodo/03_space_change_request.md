# 03 トラッカーダッシュボードの改修依頼（artifact.edit に渡す依頼文）

- 対象 Space：トラッカーダッシュボード（slug: `toratukadatusiyubodo`）
- 適用方法：もちが `artifact.edit` に、下の「依頼文の全文」をそのまま渡す（1回の依頼にまとめてある。作り直しを2回に分けないため）。
- 依頼文のあとに、もち向けの「適用後の確認手順」を置いた。

この依頼文は、2026-11-22時点のプログラム（`server/src/actions.ts` 158行、`server/src/schema.ts` 14行、`client/src/App.tsx` 617行、`DATA-PLAN.md`）を読んで書いています。

---

## 依頼文の全文

````text
トラッカーダッシュボード（slug: toratukadatusiyubodo）の改修をお願いします。中身は2つです。(1) サーバー側に「部分更新」の操作を足す、(2) 画面に情報の新しさを出す。1回の改修でまとめて実装してください。

■ 背景（なぜ変えるか）
- 今は書き手が「今の全体を読む（getSnapshot）→ 一部を書き換える → 全体を保存し直す（saveSnapshot）」というやり方をしており、2026-11-17・11-20・11-22 に、書き換えるつもりのない欄（予定の補足欄・予定の並び順）が変わってしまう事故が起きた。
- 2026-11-22 に、書き手はメインのエージェント（もち）1人にし、欄ごとの部分更新で反映すると決まった。予定・進行中・ジョブ一覧は週1回の棚卸しで更新する。
- 「最終更新」は部分的な更新のたびに新しくなるが、週1回しか更新しない欄や手で更新する欄の古さが画面から分からない。日付の過ぎた予定も先頭に残り続ける。

■ 1. データの形の追加（後方互換にすること）
保存済みの31行（2026-11-11〜11-22）は、そのまま読めなければならない。新しい項目はすべて「省略可・既定値あり」にする。既存の項目の名前・型・意味は変えない。
1-1. services の各要素と machines の各要素に、cadence を追加する。
  - 型：z.enum(["daily", "manual"]).default("daily")
  - 意味："daily" は毎日更新される欄（古くなったら警告色にする）。"manual" は手で時々更新する欄（古さは表示するが警告色にしない）。
1-2. スナップショットの最上位に sectionUpdatedAt を追加する。
  - 型：z.object({ githubNote: z.string().optional(), upcomingTasks: z.string().optional(), ongoingItems: z.string().optional(), recurringJobs: z.string().optional() }).default({})
  - 意味：その欄の情報がいつ時点のものか（ISO 8601、+09:00 などの時差付き）。services と machines は各要素の checkedAt が同じ役目を持つので、ここには入れない。
1-3. 既存の usageWindows の .default([]) はそのまま残す（1〜7行目の古いスナップショットには usageWindows がない）。

■ 2. getSnapshot の変更（読めないデータと、まだ無いデータを区別する）
- 応答に status（"ok" | "empty" | "invalid"）、snapshotId（number | null）、invalidReason（string | null）を追加する。既存の found・savedAt・snapshot はそのまま残す。
  - 行が1つもない：found=false, status="empty", savedAt=null, snapshot=null, snapshotId=null, invalidReason=null
  - 最新の行が JSON として読めない、または形の検査（zod）に通らない：found=false, status="invalid", savedAt=その行の保存時刻, snapshot=null, snapshotId=その行の id, invalidReason=検査で引っかかった場所の短い説明（例 "services[2].checkedAt: Required"。200字以内）
  - 正常：found=true, status="ok", snapshotId=その行の id, invalidReason=null
- 壊れた最新行の代わりに、1つ前の正常な行を黙って返すことはしない（復旧用には 3-8 の別の操作を使う）。

■ 3. 部分更新の操作（新設。サーバー側で完結させる）
次の7つの部分更新の操作（3-1〜3-7）と、復旧用の読み取り専用の操作1つ（3-8）を追加する。どの部分更新の操作も、呼び出し側が全体を読み書きする必要がないようにする。
3-1. updateService：AIサービスのカード1枚だけを変える
  - 入力：{ name: string, patch: { status?, statusLabel?, lines?, meterPercent?（number|null）, meterLabel?, usageWindows?, cadence?, checkedAt? }, allowCreate?: boolean（既定 false）, allowOlder?: boolean（既定 false） }
  - name は既存カードの name と完全一致で探す（前後の空白も含めて一致。あいまい一致はしない）。
  - patch に書いた項目だけを変え、書いていない項目は元の値のまま残す。カードの並び順は変えない。name は変えられない（patch に name を入れたら入力エラー）。
  - status・statusLabel・lines・meterPercent・meterLabel・usageWindows のどれかを変えるときは、checkedAt を必須にする（無ければエラー MISSING_CHECKED_AT）。cadence だけを変えるときは checkedAt を省略でき、その場合 checkedAt は元のまま。
  - allowCreate=true のときだけ、見つからない name を末尾に新しいカードとして追加できる。その場合 patch に status・statusLabel・lines・meterPercent・meterLabel・checkedAt がすべて必要（無ければエラー INCOMPLETE_NEW_ITEM）。usageWindows は省略時 []、cadence は省略時 "daily"。
3-2. removeService：AIサービスのカード1枚を取り除く
  - 入力：{ name: string }。完全一致で1枚だけ取り除く。見つからなければエラー NOT_FOUND。
3-3. updateMachine：リソース欄のマシン1台だけを変える
  - 入力：{ name: string, patch: { status?, statusLabel?, specs?, note?, cadence?, metrics?, metricUpdates?, checkedAt? }, allowOlder?: boolean（既定 false） }
  - name は完全一致。マシンの追加はこの操作ではできない（見つからなければ NOT_FOUND）。
  - metrics は計測値の配列を丸ごと置き換える。metricUpdates は [{ label, valueText, percent, detail }] の配列で、label が一致する計測値だけを置き換え、他の計測値と並び順は残す。metricUpdates の label が今の metrics に無ければエラー METRIC_NOT_FOUND（黙って追加しない）。metrics と metricUpdates を同時に指定したら入力エラー。
  - status・statusLabel・specs・note・metrics・metricUpdates のどれかを変えるときは checkedAt 必須（MISSING_CHECKED_AT）。cadence だけなら省略可。
3-4. updateGithubNote：GitHubメモだけを変える
  - 入力：{ text: string（空文字は不可）, asOf?: string, allowOlder?: boolean（既定 false） }
  - githubNote を text にし、sectionUpdatedAt.githubNote を asOf（省略時はサーバーの現在時刻）にする。
3-5. replaceUpcomingTasks：期限の近い予定の一覧だけを丸ごと差し替える
  - 入力：{ tasks: taskSchema の配列（0件も可）, asOf?: string, allowOlder?: boolean }
  - 各 task の date は実在する YYYY-MM-DD であること（違えばエラー BAD_DATE）。同じ date と同じ title の組が2つあればエラー DUPLICATE_ITEM。
  - 保存前に date の昇順に並べ替える。同じ date の中は入力の順を保つ（安定な並べ替え）。
  - upcomingTasks を差し替え、sectionUpdatedAt.upcomingTasks を asOf（省略時は現在時刻）にする。
3-6. replaceOngoingItems：進行中の項目の一覧だけを丸ごと差し替える
  - 入力：{ items: string の配列（各要素は空文字不可。0件も可）, asOf?: string, allowOlder?: boolean }
  - ongoingItems を差し替え、sectionUpdatedAt.ongoingItems を asOf（省略時は現在時刻）にする。
3-7. replaceRecurringJobs：定期ジョブの一覧だけを丸ごと差し替える
  - 入力：{ jobs: jobSchema の配列（0件も可）, asOf?: string, allowOlder?: boolean }
  - name が空、または同じ name が2つあればエラー（それぞれ BAD_REQUEST、DUPLICATE_ITEM）。並び順は入力のまま。
  - recurringJobs を差し替え、sectionUpdatedAt.recurringJobs を asOf（省略時は現在時刻）にする。
3-8. getLastValidSnapshot：最後に正しく読めたスナップショットを返す（読み取り専用・復旧用）
  - 入力：{}。行を新しい順に見て、JSONとして読めて形の検査に通る最初の行を返す。応答：{ found: boolean, snapshotId: number|null, savedAt: string|null, snapshot: snapshot|null, skippedInvalidRows: number（それより新しい壊れた行の数） }。
  - 何も保存しない。画面からは使わない。最新の行が壊れたときに、エージェントがデータベースを直接開かずに復旧の元を取り出すためのもの。

■ 4. 部分更新の共通の決まり（3-1〜3-7 の7つに適用）
4-1. サーバーの中で「最新の行を読む → 対象の欄だけ変える → 新しい行として追加保存する」を行う。既存の行は書き換えない・消さない（今の saveSnapshot と同じく、保存のたびに1行増える）。可能なら読みと追加を1つのトランザクションにする。
4-2. 最新の行が無いときはエラー NO_SNAPSHOT。最新の行が読めない（JSONとして壊れている、または形の検査に通らない）ときはエラー LATEST_INVALID とし、何も保存しない。壊れたデータの上に部分更新を積まない。この場合の復旧は、エージェントが getLastValidSnapshot で元を取り出し、saveSnapshot で正しい全体を保存し直して行う。
4-3. 対象が見つからない（name が一致しない）ときは、黙って追加せずエラー NOT_FOUND にする（updateService の allowCreate=true のときだけ例外）。同じ name の要素が2つ以上あるときはエラー DUPLICATE_NAME にし、何も保存しない。
4-4. 時刻（checkedAt・asOf）は、時差付きの ISO 8601（末尾が +09:00 などの時差、または Z）で、日時として解釈できるものだけを受け付ける。違えばエラー BAD_TIME。
4-5. 古い値で新しい値を上書きしない：渡された checkedAt（サービス・マシン）または asOf（各欄）が、今保存されている同じカード・同じ欄の時刻より古いときは、エラー OLDER_THAN_CURRENT にして保存しない。allowOlder=true のときだけ上書きを許す。同じ時刻は許す。今の欄に時刻が無い（sectionUpdatedAt に記録がない）ときは比べずに受け付ける。
4-6. 保存する前に、変更後のスナップショット全体を snapshotSchema で検査し、通らなければ保存しない（エラー VALIDATION_FAILED）。
4-7. 最上位の capturedAt は、部分更新のたびにサーバーの現在時刻（日本時間、例 "2026-11-23T08:20:05+09:00" の形）にする（「最後に保存した時刻」という意味。古い画面との互換のため残す）。各欄の情報の新しさは、capturedAt ではなく 1-1・1-2 の時刻（と各カードの checkedAt）で表す。
4-8. 成功時の応答：{ ok: true, savedAt: 保存時刻, snapshotId: 新しい行の id, previousSnapshotId: 元にした行の id, changed: 変えた対象を示す短い文字列（例 "service:Claude Pro"、"machine:Muse の VM"、"githubNote"、"upcomingTasks"） }。保存後は今の saveSnapshot と同じく ctx.invalidateQueries() を呼ぶ。
4-9. 失敗はすべて「操作の失敗」として返す（ok を返さない）。エラーの文言は「コード: 日本語の説明」の形にする（例 "NOT_FOUND: サービス「Claude Pro2」が見つかりません"）。失敗時は1行も保存しない。
4-10. 各操作の description（説明文）には、何を変えて何を変えないか、上の決まりの要点、を英語か日本語で書く。

■ 5. saveSnapshot（全体保存）
- 操作そのものと入力の形は変えずに残す（新しい項目は省略可）。
- description を次の趣旨に書き換える：「スナップショット全体を保存する。通常の更新には使わない。各欄の更新は部分更新の操作（updateService・removeService・updateMachine・updateGithubNote・replaceUpcomingTasks・replaceOngoingItems・replaceRecurringJobs）を使う。全体保存は、データが壊れて部分更新が LATEST_INVALID で拒否されたときの復旧と、初回の保存にだけ使う。」

■ 6. 画面の改善
6-1. 欄ごとの「◯日前の情報」表示と、古い欄の色
  - 「何日前か」は日本時間（Asia/Tokyo）の暦日で数える。時刻の日付（日本時間）と今日（日本時間）の差。0 なら「今日の情報」、1 なら「1日前の情報」、n なら「n日前の情報」。時刻が未来（時計のずれ）なら「今日の情報」。今の daysUntil と同じ考え方の関数を共通で使う。
  - 欄ごとの「古い」とみなす基準（この表を定数として1か所にまとめる）：
    - AIサービスのカード（cadence "daily"）：そのカードの checkedAt が2日前以上
    - リソースのマシンのカード（cadence "daily"）：そのカードの checkedAt が2日前以上
    - cadence "manual" のカード：古さを表示するだけで、警告色にしない。年齢の後に「（手動更新）」と添える
    - GitHubメモ：sectionUpdatedAt.githubNote が2日前以上
    - 期限の近い予定・進行中の主な項目・定期ジョブ：sectionUpdatedAt のそれぞれが8日前以上（週1回の更新のため）
    - sectionUpdatedAt に記録が無い欄は「更新日時の記録なし」と表示し、古い欄と同じ扱い（警告色）にする
  - 表示の場所：
    - AIサービス・マシンのカード：今ある「最終確認 2026/11/22 08:13」の行の後ろに「・今日の情報」のように続ける。
    - 予定・進行中・定期ジョブ・GitHubメモ：それぞれの枠の右上（見出しがある枠は見出しの右側）に「◯日前の情報」を小さく出す。予定の枠には今は見出しがないので、SectionTitle「タスク状況（期限が近い順）」の right に出す。
  - 古い欄の見た目：その欄の枠線を警告色（var(--warn)）にし、「◯日前の情報」の文字を警告色の太字にして、前に「古い情報・」と付ける（例「古い情報・9日前の情報」）。色だけに頼らず文字でも分かるようにする。
6-2. 上部の「最終更新」の置き換え
  - 今の「最終更新 {capturedAt}」をやめ、次の2行にする。
    - 1行目（大きい文字）：古い欄が1つも無いときは「すべての欄が新しい状態です」（通常色）。古い欄があるときは「古い欄があります：{欄の名前を最大3つ、読点区切り}（最も古いもので{n}日前）」を警告色で出し、4つ以上あるときは「ほか{m}欄」を付ける。欄の名前は「Claude Pro」「Muse の VM」「GitHubメモ」「予定」「進行中」「定期ジョブ」のように、カード名または欄の短い名前を使う。記録が無い欄は「（記録なし）」と添える。cadence "manual" のカードはここに数えない。
    - 2行目（小さい文字）：「最終保存 {getSnapshot の savedAt を 2026/11/23 09:14 の形で}」
  - 「再読み込み」ボタンは今のまま残す。ページ最下部の「スナップショット保存 …」の行も今のまま残す。
6-3. 日付の過ぎた予定
  - 日本時間の今日より前の date を持つ予定（date < 今日）を「期限を過ぎた予定」とする。今日の予定は今までどおり「今日」の札を付けて主な一覧に残す。
  - 主な一覧には、今日以降の予定だけを出す（並びはデータの順のまま）。
  - 期限を過ぎた予定が1件以上あるときは、主な一覧の下に「期限を過ぎた予定 {N}件」という折りたたみの行を出す。初期状態は閉じておき、押すと開いて、その予定を薄い文字で並べ、日付の下の札の位置に「期限切れ」の札（var(--bad) の枠線と文字）を付ける。開閉の状態は保存しなくてよい。
  - 主な一覧が0件のときは、今の「期限のあるタスクは登録されていません。」を出し、その下に折りたたみの行を出す。
  - この判定は画面を描くときの日本時間の日付で行い、データは書き換えない。
6-4. 読めないデータの表示
  - getSnapshot の status が "invalid" のときは、「スナップショットはまだ保存されていません」ではなく、「保存されたデータを読めません（データの形が崩れています）。最新の保存：{savedAt を 2026/11/23 09:14 の形で}。もちに確認を頼んでください。」を var(--bad) の色で出す。invalidReason があれば、その下に小さく等幅の文字で出す。
  - status が "empty" のときだけ、今の「スナップショットはまだ保存されていません。…」を出す。
6-5. 変えないもの
  - AIサービスの並び（Claude → Perplexity → その他）、使用率のバーとペースの判定、5時間枠を出さない決まり、Perplexity の「残り◯回」の強調表示、カードの中身の描き方は、今のまま変えない。

■ 7. DATA-PLAN.md の更新
- 「Snapshot contract addition (2026-11-23 edit: partial updates and freshness)」のような節を足し、次を書く：書き手はメインのエージェント1人で、通常の更新は部分更新の操作を使うこと（週次棚卸しのジョブが予定・進行中・定期ジョブの3欄を部分更新の操作で書くことはある）／saveSnapshot は復旧と初回専用／getLastValidSnapshot は復旧用の読み取り専用／sectionUpdatedAt と cadence の意味／欄ごとの古さの基準（毎日の欄は2日、週1回の欄は8日）／期限を過ぎた予定は画面で折りたたむだけでデータは消さないこと／LATEST_INVALID のときは部分更新が拒否されること。
- 既存の「Refresh policy」の記述は消さず、上の節が新しい決まりであることが分かるように一文添える。

■ 8. 受け入れの条件（作り手側で確かめてほしいこと）
- 既存データ（最新は 2026-11-22 09:13 保存の行）で、getSnapshot が status "ok" を返し、画面が今と同じ中身で表示される（新しい項目は既定値：cadence は "daily"、sectionUpdatedAt は {}）。
- usageWindows を持たない古い形のスナップショットも、検査に通る（.default([]) が効いている）。
- getLastValidSnapshot が、既存データでは最新の行（壊れた行が0件なので skippedInvalidRows は 0）を返す。
- 7つの部分更新の操作について：成功時は対象の欄だけが変わり、他の欄（特に upcomingTasks の並び順と note、ongoingItems、recurringJobs、githubNote、他のサービス・マシン）が1文字も変わらないこと。失敗時（NOT_FOUND、DUPLICATE_NAME、METRIC_NOT_FOUND、OLDER_THAN_CURRENT、BAD_TIME、BAD_DATE、MISSING_CHECKED_AT、LATEST_INVALID）は1行も保存されないこと。
- 画面：期限を過ぎた予定が主な一覧に出ず、「期限を過ぎた予定 N件」の折りたたみに入る。各欄に「◯日前の情報」が出る。上部に「すべての欄が新しい状態です」または「古い欄があります：…」が出る。status "invalid" のときに「データを読めません」の表示になる。
````

---

## もち向け：適用後の確認手順

改修の完了報告が来たら、次の順で確かめます。**1〜3 は書き込みをしない確認**、4 以降で初めて部分更新を使います。

### 1. 読むだけの確認（getSnapshot・getLastValidSnapshot）
- `getSnapshot` を呼び、次を確かめる。
  - `status` が `"ok"`、`found` が `true`。
  - `snapshotId` が改修前の最新（2026-11-22 時点で id 31。改修までに保存があれば、それ以降の番号）と同じ。改修でデータの行は増えないはず。
  - `snapshot` の中身が改修前と同じ（AIサービス4枚・マシン3台・予定・進行中・定期ジョブ・GitHubメモ）。新しい項目は既定値（各カードの `cadence` が `"daily"`、`sectionUpdatedAt` が `{}`）で入っている。
- `getLastValidSnapshot` を呼び、`snapshotId` が上と同じで、`skippedInvalidRows` が 0 であること。

### 2. 画面の確認（目で見る）
- 上部が「最終更新」ではなく、「古い欄があります：…」（または「すべての欄が新しい状態です」）と「最終保存 …」の2行になっている。改修直後は予定・進行中・定期ジョブ・GitHubメモに時刻の記録がないので、「古い欄があります」と出るのが正しい。
- 予定の主な一覧の先頭に、日付の過ぎた予定（2026-11-22時点のデータなら 11/18 の1件と 11/20 の3件）が**出ていない**。その下に「期限を過ぎた予定 N件」の折りたたみがあり、開くと「期限切れ」の札つきで並ぶ（N は確認する日によって変わる。11/23 に見るなら4件）。
- 各カードの「最終確認 …」の後ろに「・◯日前の情報」が出ている。Muse（11/17）と ChatGPT（11/11）、自宅ミニPC・自宅PC 2台（11/11）は、この時点では警告色になっているのが正しい（次の 4 で「手動更新」に切り替える）。
- 「スナップショットはまだ保存されていません」が出ていない。

### 3. 失敗のしかたの確認（失敗するので何も保存されない）
- `updateService` を `name: "存在しない確認用"`, `patch: { cadence: "manual" }` で呼ぶ → `NOT_FOUND` で失敗すること。
- `updateMachine` を `name: "Muse の VM"`, `patch: { metricUpdates: [{ label: "存在しない計測", valueText: "x", percent: null, detail: "" }], checkedAt: "<今の時刻+09:00>" }` で呼ぶ → `METRIC_NOT_FOUND` で失敗すること。
- 上の2回のあと `getSnapshot` の `snapshotId` が増えていないこと（何も保存されていない）。

### 4. 初回の設定（ここから実際に保存する）
1回ごとに `getSnapshot` で「対象だけが変わり、他の欄は1文字も変わっていない」ことを確かめる（`capturedAt` は変わってよい）。
- 手で更新するカードを「手動更新」にする（4回）：
  - `updateService`：`name: "Muse"`, `patch: { cadence: "manual" }`
  - `updateService`：`name: "ChatGPT"`, `patch: { cadence: "manual" }`
  - `updateMachine`：`name: "自宅ミニPC"`, `patch: { cadence: "manual" }`
  - `updateMachine`：`name: "自宅PC 2台"`, `patch: { cadence: "manual" }`
- 週1回の欄（予定・進行中・定期ジョブ）を正本から作り直す：`05_weekly_audit_addition.md` の追記手順（手順8）を、もちが1回だけ手で実行する。これで `sectionUpdatedAt` の3つに時刻が入り、日付の過ぎた予定や UQ 開通のような終わった項目、カレンダー新規検知の時刻のずれも直る。
- GitHubメモ：次に GitHub の値ファイル（`~/workspace/goals/github/hidden_files/tracker-pending/`）を反映するときに `updateGithubNote` を使えば、時刻が入る。

### 5. 仕上がりの確認
- 画面上部が「すべての欄が新しい状態です」になる（その日の朝の値の反映がまだなら、AIサービス等が「古い欄」に出るのは正しい）。
- 手で更新するカード（Muse・ChatGPT・自宅ミニPC・自宅PC 2台）は「◯日前の情報（手動更新）」と出て、警告色ではない。
- うまくいかない点があれば、`snapshotId` と画面の様子を添えて、作り手に直しを依頼する。データが壊れた（`status: "invalid"`）場合は、`04_receive_and_fallback.md` の「データが壊れたとき」に従う。
