# 2022 scale testing log

DevX teammates and teammates hacking on Sourcegraph's scaletesting set of tools.

To add an entry, just add an H2 header starting with the ISO 8601 format, a topic.
**This log should be in reverse chronological order.**

# 2022-10-17

@jhchabran @burmudar We sat down with William and listed all the things that we need to do based our recent exchanges on ScaleTesting: 

(Crossed mean a task has been created)

- We now have a GHE instance for 45 days with unlimited users
  - can be used for permissions testing
  - ~[we need to document the new instance and what comes with it](https://github.com/sourcegraph/sourcegraph/issues/43032)~
  - ~we need to have a calendar alert about it expiration date~
  - we need to investigate how to populate stuff in there
    - ~[declarative way of doing so?](https://github.com/sourcegraph/sourcegraph/issues/43045)~
      - ~[identity provider?](https://github.com/sourcegraph/sourcegraph/issues/43044)~
- GHE Feeder: Update to work with Gitlab/Bitbucket
- ~Update Scaletesting to Sourcegraph 4.0~
  - We're already running 4.0. 
- ~[Poc pipeline for Scaletesting](https://github.com/sourcegraph/sourcegraph/issues/43042)~
- Batch changes
  - Ask them what did they get from talking about this last week internally
  - ~[Need to review what Randell produced as an example scenario](https://github.com/sourcegraph/sourcegraph/issues/43043)~
    - @jhchabran: it's a really intersting doc, must read.
  - Need to be able to create repos across multiple code hosts
  - Check how it's going with the executors
- Frontend scale-testing
  - Creating notebooks on the fly
  - ~[Look into the frontend metrics](https://github.com/sourcegraph/sourcegraph/issues/42493#issuecomment-1280828981)~
  - ~[Have QH looking to the priority of FE / vs code hosts](https://sourcegraph.slack.com/archives/C02MWRMAFR8/p1666012184759779)~
    - Bet scope is challenged?
- Meet with the Code Insights team
- Meet with the Cloud Team

# 2022-10-11

@jhchabran It took me a while to finally answer Randell needs for 10k repos with write access, mostly because I got suprised by ghe-feeder. So to avoid repeating the same mistake:

- ghe-feeder will silently put the repos under the user organization if it cannot create the organization you asked for.
  - this makes it easy to mess things up, because the importing process stopped and you're trying to resume, but because the org exists, it will fail to create it.
  - also, because of this, you can end up with a weird situation where you're you're importing repos in a given organization but some of them will fail because they're already present under the user organization.
- if you want to start importing again after having deleted everything, remember to remove `feeder.database` otherwise it may skip everything that has been previously imported.

I think we should patch ghe-feeder to make it more reliable and predictable.

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


