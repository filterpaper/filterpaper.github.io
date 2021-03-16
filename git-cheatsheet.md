# Git cheatsheet
https://docs.qmk.fm/#/newbs_git_using_your_master_branch

# Branching git commits
See [managing branches](https://github.com/Kunena/Kunena-Forum/wiki/Create-a-new-branch-with-git-and-manage-branches) for details. Create a new branch just for specific changes (skip if branch is already on repo) and push to own repo:
```
git checkout -b <new_branch>
git push origin <new_branch>
```
Commit and push changes of branch to github:
```
git commit -a -m "comments"
git push origin [new_branch]
```

# Setup QMK repository as upstream to sync
One time setup of the original repository as a remote "upstream":
```
git remote add upstream https://github.com/qmk/qmk_firmware.git
```
## Merge upstream/master with fork (origin) master
```
git fetch upstream
git checkout master
git merge upstream/master
git push origin master
```
## Sync branch with updated (origin) master
### Using merge
```
git checkout <branch>
git merge master
git push origin <branch>
```
### Using rebase
```
git checkout <branch>
git rebase -i master
git push origin <branch>
```
## Overwrite forked (origin) master commits with upstream/master
If (accidental) master commits are to be discarded, use reset over write QMK fork master with upstream/master
```
git fetch upstream
git checkout master
git reset --hard upstream/master
git push origin master --force
```

# Pruning branches
After a branch PR is accepted into master, it can be deleted at the end of the PR. Sync the forked master with upstream, confirm that the merged and deleted branch can be pruned with dry run, then actually prune it:
```
git remote prune --dry-run origin
git remote prune origin
```
Delete that branch on locally
```
git branch -d <pruned branch>
```

# Reverting commits
Committed mistakes can be reverted using reset. Find the commit ID to reset to that point. Checkout to the right branch or master, reset to that ID, run log to confirm position, then force push back github to reset repo to that previous ID:
```
git reset --hard <commit ID>
git log
git push origin <branch/master> --force
```
