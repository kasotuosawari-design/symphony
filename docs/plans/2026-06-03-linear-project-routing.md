# Linear Project Routing for No-Project Issues

> Status: proposal / planning. Implementation is sequenced but not started.
> Owner repo for execution: this fork (`Symphony-Ryo-Lab`, Elixir runtime).

## Problem

GitHub Issue を起点に Linear Issue を自動生成する経路で、生成直後の Linear Issue が
`No project` のまま残ることがある(例: `LAB-297`)。Symphony runtime のポーラは
**Project でフィルタして claim する**ため、`No project` の Issue は構造的に runner から
不可視になり、開始フローにも到達しない。

これは「Project が無いと拾えない / 拾えないと Project 補正も走らない」という鶏卵問題。

### Root cause (grounded)

ポーリングクエリが Project スラッグで Issue を絞り込んでいる:

```graphql
# elixir/lib/symphony_elixir/linear/client.ex  (@query)
issues(filter: {project: {slugId: {eq: $projectSlug}}, state: {name: {in: $stateNames}}}, ...)
```

`fetch_candidate_issues/0` は `tracker.project_slug` 必須で、未設定 Issue を返す経路が無い
(`elixir/lib/symphony_elixir/linear/client.ex`, `tracker.ex`)。
`No project` の Issue はこのクエリ結果に現れない。

## Goal

`No project` で取り残された Issue を、runtime が **claim 直前に reconcile(Project 付与)** し、
そのまま正規の claim 経路へ載せる。push 型(Issue 作成時に必ず同期させる)ではなく、
**消費境界での pull 型 self-healing** で解く。webhook の取りこぼしを構造的に吸収する。

## Responsibility split (合意済み)

実装は3層で分離し、推定ロジックを1箇所に集約して drift を防ぐ。

| 層 | 役割 | やらないこと |
|---|---|---|
| `auto_template`(定型文生成) | Slack テンプレに「repo名 = Linear Project名」ルールを記載。起票時に **repo hint を Issue body/label に載せる(素材を渡すだけ)** | Project 推定・付与の判断、同期の強制 |
| `Symphony-Ryo`(Python policy/state) | human merge policy、Linear/GitHub audit、safety gate。routing の **policy 上の所有者** | — |
| `Symphony-Ryo-Lab`(この Elixir runtime) | repo→Project マッピングを **宣言的 config として保持・実行**。`No project` の発見と claim 前 reconcile | — |

### 推定ルールの所在(決定事項)

repo→Project の推定は **`WORKFLOW.md` の宣言的ルーティング設定**として表現し、
Elixir runtime がそれを読んで実行する。

- Elixir runtime から Python ロジックを実行時に呼ぶ「クロス言語ランタイム結合」は採らない
  (脆く、独自路線になる)。
- SoT は version 管理された `WORKFLOW.md` の routing 設定ただ1つ。
- 結果として、`auto_template` 側に PR #436 / commit 78596bc で入った Project 推定ロジック
  (`automation._ensure_linear_project` 等)は **冗長化するため退役候補**。本計画の着地後に
  別 PR で縮退し、`auto_template` は repo hint 出力のみ残す。

## Inference contract (repo → project)

優先順位付きの候補列を作り、先頭から既存 Project にマッチしたものを採用する:

1. **明示 map**: `repo (owner/name or name)` → `project_slug` の宣言的マッピング
2. **repo名フォールバック**: repo 名と同名の Project を探索
3. **既定フォールバック**: 設定された default project

repo の取得元は Linear Issue に紐づく GitHub 情報(attachment / branchName / description 内の
repo hint)。どれも取れない場合は default にも match しなければ reconcile せず、
Issue にコメントを残して human に委ねる(silent な誤割当を作らない)。

## Design

### Config additions (`WORKFLOW.md` / `config/schema.ex`)

`tracker` セクションに後方互換な任意フィールドを追加する。未設定なら現状の挙動を維持。

```yaml
tracker:
  kind: linear
  project_slug: "symphony-ryo"      # 既存: 既定 / フォールバック先
  team_key: "LAB"                    # 追加: No-project 発見クエリのスコープ
  active_states: [Todo]
  project_routing:                   # 追加: 明示 map(優先順位 1)
    enabled: true
    fallback_to_repo_name: true      # 優先順位 2
    map:
      "kasotuosawari-design/Symphony-Ryo": "symphony-ryo"
      # owner/name もしくは name で指定
```

- `config/schema.ex` の `Tracker` embedded schema に `team_key`, `project_routing` を追加。
- 既存テスト・既存 `WORKFLOW.md` は無改変で通ること(全フィールド任意 + デフォルト)。

### Discovery query (No-project)

Project フィルタを外し、`team_key` + `active_states`(+ routing label があればそれ)で
スコープした「reconcile 候補」取得クエリを追加する。全 workspace を引かないため team scope は必須。

```graphql
# 新規: fetch_unprojected_candidates
issues(filter: {
  team: {key: {eq: $teamKey}},
  project: {null: true},
  state: {name: {in: $stateNames}}
}, first: $first, after: $after) { nodes { id identifier url description branchName ... } }
```

- `Tracker` behaviour に `fetch_unprojected_candidates/0` を追加
  (`tracker.ex`, `linear/adapter.ex`, `linear/client.ex`, memory adapter)。
- `team_key` 未設定時は no-op(空リスト)で安全側に倒す。

### Reconcile (project 付与)

1. 候補 Issue ごとに repo hint を抽出(attachment / branchName / description)。
2. inference contract で project 候補列を生成。
3. 既存 Project に解決できた先頭候補へ `issueUpdate(input: {projectId})` で付与
   (`update_issue_project` 相当の mutation を adapter に追加)。
4. 解決不能なら Issue にコメントして skip(誤割当を作らない)。

冪等性: 既に正しい Project の Issue は変更しない。reconcile 後は通常の
`fetch_candidate_issues/0` が拾えるようになる。

### Orchestrator wiring

`orchestrator.ex` のポーリングサイクルで、通常 poll の **前段**に reconcile を1パス挟む:

```
poll tick
  └─ reconcile_unprojected()        # 追加: 発見 → 推定 → 付与(best-effort, 失敗は warn)
  └─ fetch_candidate_issues()       # 既存: project でフィルタ済みを claim
```

reconcile は best-effort。失敗してもサイクルを止めない(ログ + 次サイクルで再試行)。

## Out of scope

- `auto_template` 側推定ロジックの撤去(別 PR。本計画着地後)。
- Linear webhook ハンドラの新設(将来の最適化。reconcile の速度改善であって正しさの保証ではない)。
- 自動 merge / 自動 Done(human gate のまま不変)。

## Alternatives considered

- **push 型(GitHub Issue 作成時に Linear Issue 生成 + Project 付与を強制)**: 失敗経路と
  タイミング依存が増え脆い。`Refs LAB-XXX` の逆抽出はミラー生成前に webhook が来ると取れない。不採用。
- **推定を `auto_template` に残す**: 定型文生成ツールの責務を越える。runner が consumer なので
  消費側に reconcile を置く方が自己修復的。不採用。
- **Elixir runtime から Python policy を実行時呼び出し**: クロス言語ランタイム結合で脆く、
  独自路線。宣言的 config に倒す。不採用。

## Implementation steps (sequenced)

1. `config/schema.ex` に `tracker.team_key` と `tracker.project_routing` を後方互換追加 + テスト。
2. `Tracker` behaviour + Linear/memory adapter に `fetch_unprojected_candidates/0` と
   `update_issue_project/2` を追加 + テスト。
3. repo hint 抽出 + inference contract(明示 map → repo名 → default)モジュール + 単体テスト。
4. reconcile パス実装(冪等・解決不能はコメント skip)+ テスト。
5. `orchestrator.ex` のポーリング前段に reconcile を wiring + 統合テスト。
6. `WORKFLOW.md` に routing 設定例とコメントを追記。
7. runbook(`docs/symphony-ryo-lab-runbook.md`)に reconcile の動作と無効化方法を追記。

## Test plan

- `make -C elixir all`(format / credo / dialyzer / test)。
- memory adapter で `No project` → reconcile → claim の通し統合テスト。
- routing 未設定の既存 `WORKFLOW.md` で挙動不変(回帰なし)を確認。
- 解決不能ケースで誤割当せずコメント skip することを確認。
