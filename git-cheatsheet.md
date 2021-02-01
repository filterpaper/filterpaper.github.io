# Git cheatsheet
https://docs.qmk.fm/#/newbs_git_using_your_master_branch

# Branching git commits
See [managing branches](https://github.com/Kunena/Kunena-Forum/wiki/Create-a-new-branch-with-git-and-manage-branches) for details.
Create a new branch just for specific changes (skip if branch is already on repo) and push to own repo:
```
git checkout -b [new_branch]
git push origin [new_branch]
```
Switch to that branch to make changes:
```
git checkout [new_branch]
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
## Sync forked QMK master with QMK upstream
```
git fetch upstream
git checkout master
git merge upstream/master
git push origin master
```
## Overwrite forked master commits with upstream
If (accidental) master commits are to be discarded, use reset instead of merge:
```
git fetch upstream
git checkout master
git reset --hard upstream/master
git push origin master --force
```
## Sync forked QMK branch with locally updated forked QMK master
```
git checkout <branch>
git merge master
git push origin <branch>
```

# Pruning branches
After a branch PR is accepted into master, it can be deleted at the end of the PR.
Sync the forked master with upstream, confirm that the merged and deleted branch can
be pruned with dry run, then actually prune it:
```
git remote prune --dry-run origin
git remote prune origin
```
Delete that branch on locally
```
git branch -d <pruned branch>
```

# Reverting commits
Committed mistakes can be reverted using reset. Find the commit ID to reset to that
point. Checkout to the right branch or master, reset to that ID, run log to confirm
position, then force push back github to reset repo to that previous ID:
```
git reset --hard <commit ID>
git log
git push origin <branch/master> --force
```
