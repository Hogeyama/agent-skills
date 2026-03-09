---
name: git-rewrite
description: Rewrite git history by modifying existing commits via interactive rebase. Use this skill whenever the user wants to edit, fix, or clean up past commits — for example removing lines from old commits, splitting commits, rewriting commit messages across multiple commits, or making surgical changes to specific commits in history. Triggers on phrases like "rebase", "edit commit", "fix that commit", "clean up history", "remove from old commit", "amend past commit", or any request that implies changing commits that aren't HEAD.
---

# Git History Rewrite via `git rebase -i`

## `GIT_SEQUENCE_EDITOR` + `sed -E` パターン

`git rebase -i` のtodoリストは `rebase.abbreviateCommands` 設定次第で `pick` または `p` になる。
**必ず `sed -E` で `^(p|pick)` を使い、両方にマッチさせること。**

```bash
# edit (内容変更): 停止 → 修正 → git commit --amend --no-edit → git rebase --continue
GIT_SEQUENCE_EDITOR="sed -i -E \
  -e 's/^(p|pick) (<hash1>)/edit \2/' \
  -e 's/^(p|pick) (<hash2>)/edit \2/'" \
  git rebase -i <base>

# reword (メッセージ変更のみ)
GIT_SEQUENCE_EDITOR="sed -i -E 's/^(p|pick) (<hash>)/reword \2/'" git rebase -i <base>

# drop (コミット削除)
GIT_SEQUENCE_EDITOR="sed -i -E 's/^(p|pick) (<hash>)/drop \2/'" git rebase -i <base>
```

## 完了後: `git range-diff` の表示

リベースが成功したら、**必ず** `git range-diff` で旧コミットと新コミットの差分をユーザーに提示する。

リベース開始前に旧 HEAD のコミットハッシュを変数等で控えておき、完了後に以下のように比較する。

```bash
# リベース開始前
OLD_HEAD=$(git rev-parse HEAD)

# ... リベース実行 ...

# リベース完了後
git --no-pager range-diff <base>..$OLD_HEAD <base>..HEAD
```

* `<base>` はリベースの起点コミット（`git rebase -i <base>` で指定したもの）。
* `$OLD_HEAD` はリベース開始前に控えた旧 HEAD。
* これにより各コミットごとに何が変わったかが一目でわかる。

## Stale rebase の掃除

"rebase-merge directory" エラーが出たら:

```bash
git rebase --abort 2>/dev/null; rm -fr .git/rebase-merge 2>/dev/null
```
