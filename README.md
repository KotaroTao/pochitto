# pochitto

歯科医院向けLINEアカウント「ポチッとデンタル」（登録歯科医師 約3,000名）を起点に、
**お悩み相談 → 情報提供 → 最適サービス紹介**をクリックだけで完結させ、顧客満足度と自社利益を最大化する仕組みの設計リポジトリ。

設計ドキュメントは [`docs/`](./docs/README.md) を参照。

- [ルーティング優先度マスター](./docs/01-routing-priority-master.md)（中核IP）
- [収益シミュレーション](./docs/02-revenue-simulation.md)
- [プッシュ配信トリガー設計](./docs/03-push-segmentation-triggers.md)
- [実装手順書（やさしい版）](./docs/04-implementation-steps.md)
- [医院カルテ 質問項目＆タグ設計](./docs/05-clinic-profile-questions.md)
- [送客システム設計（ワンクリック送客＋リアルタイム共有）](./docs/06-referral-system.md)
- [routing-priority-master.csv](./data/routing-priority-master.csv)（Lステップ取込用）