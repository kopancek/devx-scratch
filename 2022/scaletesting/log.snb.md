# 2022 scale testing log

DevX teammates and teammates hacking on Sourcegraph's scaletesting set of tools.

To add an entry, just add an H2 header starting with the ISO 8601 format, a topic.
**This log should be in reverse chronological order.**

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

