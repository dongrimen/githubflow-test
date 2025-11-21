# GitflowからGitHub Flowへの移行手順書

## 1. 目的と概要
開発スピードの向上と管理コストの削減を目的に、現在のGitflowからGitHub Flowベースの運用へ移行します。

### 最大の変更点
* **develop ブランチを廃止します。** 常に `main` ブランチが最新・正となります。
* ローカルでの `git flow` コマンドは**使用禁止**となります。
* **すべての機能リリースにおいて、一時的な `release/*` ブランチを作成し、必ず経由します。**
* マージ操作は原則としてGitHub上の **プルリクエスト（PR）** で行います。

[Image of gitflow vs github flow diagram]

---

## 2. ブランチ運用の新ルール

### 基本原則

| 項目 | 旧 (Gitflow) | 新 (GitHub Flow 改) |
| :--- | :--- | :--- |
| **親ブランチ** | `develop` | `main` |
| **作業ブランチ作成元** | `develop` | `main` |
| **機能ブランチのマージ先** | `develop` | **`release/*` ブランチ** |
| **`feature/*` のマージ方法** | ローカルで `git flow finish` | GitHub上で **「Squash and merge」** |
| **`release/*` のマージ方法** | `main` へ通常マージ | GitHub上で **「Create a merge commit」** |
| **リリースプロセス** | `release` ブランチを作成 | **すべての機能は** `release/*` を経由し、本番リリース |

### マージボタンの使い分け（重要）
GitHubのPRマージボタンは、用途によって明確に使い分けます。

#### A. Squash and merge (スカッシュマージ)
* **用途:** 通常の機能開発 (`feature/*` → **`release/*` ブランチ**へマージ)
* **理由:** 開発中の細かいコミット履歴を1つにまとめ、リリースブランチの履歴を綺麗に保つため。

#### B. Create a merge commit (マージコミット作成)
* **用途:** リリース作業 (**`release/*`** → **`main`** へマージ)
* **理由:** 「リリース」というイベントと、そこに含まれる機能の履歴を保持するため。（**通常マージ**）

---

## 3. 移行作業（管理者のみ・移行日当日）
> **Note:** 移行日に管理者が一度だけ実行します。

### ① developブランチの統合
現在の `develop` の内容を `main` にマージし、プッシュします。

```bash
git checkout main
git merge develop
git push origin main
```

### ② developブランチの削除・凍結

今後誤って使用しないよう削除します。

```bash
git push origin --delete develop  # リモート削除
git branch -d develop             # ローカル削除
```

### ③ GitHub設定変更

1.  **Default branch:** `Settings` > `General` > `Default branch` を `main` に変更。
2.  **Branch protection:** `Settings` > `Branches` > `Branch protection rules` で `main` を保護（直接プッシュ禁止、PR必須など）。

-----

## 4. 機能開発から本番リリースまでのフロー（全メンバー）

単独の機能であっても、複数機能の同時リリースであっても、以下の手順で**必ずリリースブランチを経由**します。

### ① 開発開始

必ず `main` を最新にしてからブランチを作成してください。

```bash
git checkout main
git pull origin main
git checkout -b feature/機能名
```

### ② 開発・プッシュ

コミットは細かく行って構いません。完了したらリモートへプッシュします。

```bash
git add .
git commit -m "機能Aの実装（途中）"
git push origin feature/機能名
```

### ③ リリース用ブランチ（箱）の作成（管理者）

リリース対象の機能が揃ったら、管理者が `main` からリリース用のブランチを作成します。

```bash
git checkout main
git pull
git checkout -b release/v1.1.0
git push origin release/v1.1.0
```

[Image of git release branch strategy diagram]

### ④ 機能ブランチのPR向き先を変更

今回リリースしたい機能のPR（機能A, B, Cなど）の作成者は、PRの編集画面で **Base branch** を `main` から **`release/v1.1.0`** に変更します。

### ⑤ リリースブランチへのマージ（検証環境へのデプロイ）

各PRを `release/v1.1.0` に対して **「Squash and merge」** します。

* `release/v1.1.0` ブランチが検証環境（Staging）にデプロイされ、テストを実施します。
* バグが見つかった場合、`release/v1.1.0` から修正ブランチを切り、修正してPRを出します。

### ⑥ 本番リリース（mainへの統合）

テスト完了後、本番へ反映します。

1. **PR作成:** `release/v1.1.0` から `main` へのPRを作成します。  
   * タイトル例: `Release v1.1.0`
2. **マージ:** この時だけは **「Create a merge commit」** を選択します。（**通常マージ**）
3. **タグ付け:** マージ完了後、GitHubの「Releases」から新しいリリースを作成します。  
   * Choose a tag: `v1.1.0`  
   * Target: `main`  
   * `Generate release notes` をクリックして変更点を自動生成。

### ⑦ リリースブランチの削除

リリース完了後、`release/v1.1.0` ブランチは削除します。

-----

## 5. トラブルシューティング

**Q. 間違えて develop ブランチを探してしまいます。**

> **A.** 手癖を直すために、ローカルで以下のコマンドを実行してリモートの削除情報を反映させてください。  
>
> ```bash
> git fetch --prune
> ```

**Q. main にマージしたあと、バグが見つかりました。（Hotfix）**

> **A.** `main` から `hotfix/xxx` ブランチを作成し、修正してPRを出してください。  
>
> * `hotfix` のPRのマージ先は **`main`** とし、**「Squash and merge」** を使用してください。  
> * リリースブランチへの修正バックポートは、後続のリリースフローで別途検討してください。

