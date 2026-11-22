# beh — Behavioral Diff for code review

コミット・PR・Issue について、コードの変更が「どの振る舞いを、どの条件で、どう変えるか」を、コードの場所と理由つきで書き出す Claude Code のスキル。

人間のレビュワーが変更を理解するための出力を作る。バグの指摘や、変更の良し悪しの判定はしない。

## 使い方

レビューしたいリポジトリのディレクトリで、Claude Code から次のように起動する。

```
/beh #123        # PR または Issue の番号（# は省いてよい）
/beh a1b2c3d     # コミットハッシュ
/beh v1.2..v1.3  # コミットの範囲
/beh             # HEAD のコミット
```

- PR は、base との分岐点から head までの差を見る
- Issue は、その Issue を閉じた PR・本文で参照する PR・コミットメッセージで参照するコミットを集め、PR またはコミットごとに出力する
- 依存の自動更新（Dependabot・Renovate など）は、更新したパッケージと版だけを出す

## 出力

変更の目的（コミットメッセージ・PR・Issue からの引用）、変更の内容、振る舞いの変化（変更前 → 変更後、コードの場所、なぜそうなるか、確かめているテスト、注意）、変わらない振る舞い、仕様との対応、動作に影響しない変更、判断できなかったこと。

振る舞いの変化は、次の 3 つの対象で書く。

- アプリ：画面の表示と操作、HTTP の応答、保存されるデータ、外部への送信
- 開発環境：テストの実行、ビルドの道具、リンタ・型の検査
- 作業者（AI）：`CLAUDE.md`・`AGENTS.md`・`.claude/` などの変更で変わる、AI のコーディングエージェントの振る舞い

形式の詳細は `skills/beh/references/output-template.md`。

## 必要なもの

- Claude Code
- git
- GitHub の PR・Issue の番号で指定するときは `gh`（GitHub CLI）。ログイン済みであること

## 入れ方

どのリポジトリでも `/beh` で使うには、スキルのディレクトリを `~/.claude/skills/beh` に置く。

```sh
git clone <このリポジトリ> ~/Work/beh
ln -s ~/Work/beh/skills/beh ~/.claude/skills/beh
```

1 つのリポジトリだけで使うなら、そのリポジトリの `.claude/skills/beh` に置く。

## 構成

```
skills/beh/
├── SKILL.md                      手順（引数の解釈、比べる範囲、影響のたどり方、出す前の点検）
└── references/
    └── output-template.md        出力のテンプレートと書き方の決まり
```
