# kotsusaku-dashboard

コツコツサクサクの集計ダッシュボードです。

## Firestore読み取りルール

Firestoreの読み取り上限を守るため、ダッシュボードの統計表示は必ず集計済みドキュメントを低頻度で読む方針にしてください。

詳しくは [FIRESTORE_READ_POLICY.md](FIRESTORE_READ_POLICY.md) を参照してください。
