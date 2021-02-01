Copyright (C) 2021 @filterpaper

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 2 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.

# Git cheatsheet
https://docs.qmk.fm/#/newbs_git_using_your_master_branch

## Setup QMK repository as upstream
One time setup of the original repository as a remote "upstream":
```
git remote add upstream https://github.com/qmk/qmk_firmware.git
```

## Sync forked master with upstream
```
git fetch upstream
git checkout master
git merge upstream/master
git push origin master
```
### Overwrite forked master commits with upstream
If master commits are to be discarded, use reset instead of merge:
```
git fetch upstream
git checkout master
git reset --hard upstream/master
git push origin master --force
```
## Sync branch with an updated master
```
git checkout <branch>
git merge master
git push origin <branch>
```

## Commit new changes
```
git commit -a -m "<comments>"
git push
```

## Branching git commits
See [managing branches](https://github.com/Kunena/Kunena-Forum/wiki/Create-a-new-branch-with-git-and-manage-branches) for details.
Create a new branch just for keyboard or user space changes
```
git checkout -b [new_branch]
```
Push the branch on github
```
git push origin [new_branch]
```
Switch to that branch to make changes:
```
git checkout [new_branch]
```
Push changes to branch on github
```
git push origin [new_branch]
```
