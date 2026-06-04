# インシデント記録：totalStudyMin 過加算・グラフ異常

**発生日**: 2026年6月  
**解決日**: 2026年6月  
**影響**: ダッシュボード・アプリ設定画面の学習時間が約2.77倍に膨らんで表示された

---

## 何が起きたか

### 症状

| 画面 | 症状 |
|---|---|
| ダッシュボード | `totalStudyMin` が正しい値の約2.77倍で表示（例：正しくは 801,490分 → 2,220,250分 で表示） |
| アプリ設定画面 | 24h推移グラフが「0 → 急騰 → 横ばい」の不自然な形になった |
| ダッシュボード | タスクの完了タスク数が「収集中」のまま |

### 発生していた問題（3つ独立）

**① totalStudyMin の二重書き込み（主要因）**

```
【クライアント（アプリ）】
  学習セッション終了 → analytics_summary/summary.totalStudyMin += 今回の分数
                                                                   ↑ 直接加算

【サーバー（Cloud Functions）】
  毎時・手動集計 → analytics/{userId} を全件読んで合計
                → analytics_summary/summary.totalStudyMin = 合計値
                                                            ↑ 上書き
```

この2つが同じフィールドに書く設計になっており、時間が経つにつれ誤差が積み重なった。  
さらに手動再集計を実行するたびに乖離が広がった。

**② completedTasks フィールドの未集計**

Cloud Functions の集計処理に `completedTasks` の集計が含まれておらず、  
Firestore にフィールド自体が存在しないためダッシュボードが「収集中」と表示し続けた。

**③ analytics_task_trend の古いドキュメントに totalStudyMin がない**

Functions のデプロイ前に作成された `analytics_task_trend` のドキュメントには  
`totalStudyMin` フィールドが存在しなかった。  
グラフ描画コードがこれを `0` として扱ったため、`0 → 現在値` への急激な段差が発生した。

---

## なぜ発見が遅れたか

### 1. 複数の独立した問題が同時に存在していた

3つの問題が症状として混在し、どれが何の原因かの切り分けに時間がかかった。

### 2. 誤帰属（アトリビューションエラー）

「直前のダッシュボード変更が原因では」という第一印象で調査が始まったため、  
本来見るべきサーバー側（Cloud Functions）の診断が後回しになった。

### 3. 調査対象のコードが分断されていた

| 場所 | 確認できたか |
|---|---|
| `dashboard.html`（フロントエンド） | ✅ |
| Cloud Functions（集計ロジック） | ❌ 別リポジトリ・別環境 |
| アプリ本体のコード | ❌ 別リポジトリ |

根本原因がCloud Functionsにあったにもかかわらず、そのコードが見えない状態での診断になった。

### 4. 「収集中」表示がエラーの性質を隠蔽していた

```javascript
// フィールド未存在・値が0・未集計 のすべてが同じ「収集中」表示になる
setText('task-completed', completedTasks ? completedTasks + '件' : '収集中');
```

エラー・欠損・ゼロが区別できない表示設計のため、問題の性質が判断しにくかった。

### 5. データ整合性チェックの仕組みがなかった

`analytics/{userId}` の個別合計と `analytics_summary/summary.totalStudyMin` の乖離を  
検知する監視が存在せず、異常が長期間気づかれなかった。

---

## 対処

| 問題 | 対処内容 |
|---|---|
| totalStudyMin の過加算 | クライアントからの直接加算を停止。サーバー集計のみで上書きする設計に統一。Firestore の値を正しい値に手動修正。 |
| completedTasks 未集計 | Cloud Functions に completedTasks の集計処理を追加。 |
| グラフの急騰 | totalStudyMin がない古いドキュメントをグラフから除外。キャッシュキーのバージョンを更新して再取得。1点でもクラッシュしない実装に修正。 |

---

## 再発防止策

### 設計レベル

**「analytics_summary はサーバーだけが書く」ルールを徹底する**

```
✅ 正しい設計
  クライアント → analytics/{userId} のみ書く（個人記録）
  サーバー     → analytics/{userId} を読んで analytics_summary に書く

❌ やってはいけない
  クライアント → analytics_summary に直接書く（サーバーと競合する）
```

**Firestore に新フィールドを追加するときは既存ドキュメントのバックフィルも行う**

古いドキュメントに新フィールドがない場合、グラフ・集計で 0 扱いされ異常な表示になる。  
→ デプロイ時にマイグレーションスクリプトで既存ドキュメントを更新する。

### 実装レベル

**欠損・エラー・ゼロを区別して表示する**

```javascript
// Before（問題あり）
setText('task-completed', completedTasks ? completedTasks + '件' : '収集中');

// After（区別できる）
if (completedTasks === undefined) setText('task-completed', 'データなし');
else if (completedTasks === 0)    setText('task-completed', '0件');
else                               setText('task-completed', completedTasks + '件');
```

**グラフはフィールド欠損を明示的に除外する**

```javascript
// 値が存在するドキュメントだけをグラフに使う
const points = snaps.filter(s => s.totalStudyMin != null);
```

### 監視レベル

**analytics/{userId} の合計と analytics_summary の値を定期的に照合する**

乖離が一定割合（例：±10%）を超えたら Cloud Functions のログにアラートを出す。

### 診断プロセス

**複数症状が出たときは「1症状 = 1原因」に分解してから調査する**

問題が絡み合っているように見えても、まず独立した問題に切り分ける。  
「直前の変更が原因」は仮説として持つが、証拠なく確定しない。

---

## 教訓

> **「サマリーの書き込み権限は一箇所に集約する」**  
> クライアントとサーバーが同じ集計フィールドに書くと、どちらが正しい値かが  
> 曖昧になり、時間とともに乖離が広がる。サマリー系のフィールドは  
> サーバー側の集計処理だけが書く、と決めて守ることが重要。
