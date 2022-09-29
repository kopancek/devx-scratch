# 2022 scale testing log

DevX teammates and teammates hacking on Sourcegraph's scaletesting set of tools.

To add an entry, just add an H2 header starting with the ISO 8601 format, a topic.
**This log should be in reverse chronological order.**

## 2022-09-29

@jhchabran @burmudar

- We took over Manny's PR https://github.com/sourcegraph/deploy-sourcegraph-scaletesting/pull/9 and merged it
- The deployment failed because of a permission issue so we did https://github.com/sourcegraph/infrastructure/pull/3978 and applied the changes
- This fixed it. But we stumbled on issues with the git-combine pods failing because the repos were not present on the mounted volume that holds the repos
- We connected to the `src-cli` pod and created those (see https://github.com/sourcegraph/sourcegraph/tree/main/internal/cmd/git-combine) 
- We forgot to create a `bare` repo, is that a problem? 
- We add just two remotes and it's working.

## 2022-09-22 

@jhchabran @burmudar @davejrt Having fun with perforce. 

Goal is to import a git repo into perforce, while preserving the history. 

```sh
git clone some-repo
# create depot on peforce 
# add at least one change in that depot
cd some-repo 
p4 client # and check that the depot is in your view
git p4 sync //depot-name
git rebase p4/master
P4EDITOR=/usr/bin/vim +wq git p4 submit # +wq so it auto accepts each changes
```

Log: 
- I'm trying to import a smaller repo, such as genjidb/genji. It's taking ages and it's just 2000 commits. Megarepo has 700k commits. 
- Tried with my own vimconfig, about 100 commits, took 5m
- Now trying in the devx cloud instance to do the same on genjidb/genji (2k commits)
  - `gcloud compute ssh --zone "us-central1-a" "devx"  --project "davedev-sg"`
  - `cd /sg/scaletesting` 
  - `source p4.sh`
  - `cd /sg/scaletesting/repos/medium` 


