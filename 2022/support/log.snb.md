# 2022 support log

DevX support rotation log. To add an entry, just add an H2 header with ISO 8601 format. The first line should be a list of everyone involved in the entry. For ease of use and handing over issues, **this log should be in reverse chronological order**, with the most recent entry at the top.

## 2022-11-25

@jhchabran _when banging your head at walls actually reveals a big bug that has been around forever_

while I was working on https://github.com/sourcegraph/sourcegraph/pull/44795 for Erik, I was banging my head at why some file was not present 
in a volume I was mounting in docker compose in CI and finally came to the horrendous realization that none of the volumes we're mounting are ever present. 

I reached out to @davejrt for a sanity check, (because I do question my sanity often, with reason right?) and that's indeed the case. This is happening because 
as we're running the docker engine in the dind container of the agent pod/job the paths for bound volumes are not making sense to the engine, because they're evaluated 
in the dind container, not in the agent container. And that's how we end with blank folders in all these volumes we mounted. 

See https://github.com/sourcegraph/sourcegraph/issues/44816, which is pretty important to fix quickly.

@jhchabran _A tale on how the legendary @davejrt showed again his dark magic to us mere mortals_ 

As part of the above entry, I had to run the executor container which in turn also run some containers on its own. Rather than going for _dindind_, the glorious @davejrt had the idea of simply exposing 
the `DOCKER_HOST=localhost:2375` to the executor container. He said it like, hahah that might work, you know like how you say it when you now that some random crap is going to get in the way 
and mess up the glorious idea. But, if you allow me to use @davejrt's native tongue, _fucking worked_. 

A haiku about him: 

_docker is a mess_
_yet dave prevails as always_
_a true sorcerer_

## 2022-11-18

@marekweb

Fixing the smoke tests for dotcom

@jhchabran noticed that the web smoke tests were broken.  Upon investigation, the smoke tests were not working since July 2022 because the underlying implementation of the search input box changed, which broke the test's ability to fill the input to do a test search.

[I fixed the failing test](https://github.com/sourcegraph/smoke-tests/pull/43) by rewriting it to trigger a search using a query parameter instead of filling the search input box.

Notes about the web smoke tests for dotcom:

 - The smoke tests are triggered every 10 minutes in [pipeline.scheduled-smoke-tests.yaml](https://sourcegraph.sourcegraph.com/github.com/sourcegraph/deploy-sourcegraph-cloud/-/blob/.buildkite/pipeline.scheduled-smoke-tests.yaml)
 - The test is implemented in [sourcegraph/smoke-tests](https://github.com/sourcegraph/smoke-tests)
 - The test is deployed to NPM at [@sourcegraph/web-smoke-tests](https://www.npmjs.com/package/@sourcegraph/web-smoke-tests)

Potental future steps:

- We could re-evaluate if the smoke tests are valuable, given that several months went by while they were broken
- If they are in fact valuable, we could use a better alerting approach than just posting in #alerts-cloud which isn't monitored and is full of noisy alerts

## 2022-11-16
@burmudar A bit late but thought I'd log two things. Plus this is also valuable keyboard practice that I need.

### Thing 1: Sneaking in unformatted changes into main

For a while prettier was disabled on our pipelines due to its flakiness. @pjlast decided to fix it by adding a format linter to be executed as part of all the linters we execute when we do `sg lint`. In the pipeline we look at the diff to decide what linters the pipeline will execute for your build. So if your changeset touched Go files and frontend Typescript then the lint targets on your pipeline would be `go` and `client`. With @pjlast change, the `format` lint target would also be added autmatically by `sg`.

Now that we have some background, we can ask the question "How did unformatted changes sneak into main?", since we had quite a few devs say that their build failed on a file they didn't touch and it is unrelated to their change. As luck would have it, when I ran `sg lint format` on main I got a failure. I looked at git blame and found the PR where the change came in and to my surprise the lint step did run and had no failure! How could this be!? Looking at lint step for the build that ran I saw that it executed `sg lint client` which I then ran locally on the same branch as well. Low and behold it passed, but when I ran `sg lint format` it failed. Puzzled I tried a few more variations `sg lint`,`sg lint go client` and `sg lint go`. In two cases the format check did run but whenever you specified just one lint target the format lint didn't run. So in if in your PR the detected changes results in only one target like ui changes the format check would not run and your unformatted changes on your branch gets in. Ok, so we've narrowed down the condition a bit, time to go see WHY.

Digging into the `sg lint` code we can see that most of the logic is concerned with executing lint with no arguments which means execute all lint targets, and when you specify **more than one lint target**. Considering our bug happens **when we specify exactly one argument** where getting closer to the code that is misbehaving. Indeed, when looking at the this branch of code we can clearly see that there is no code to add the formatter to the linters being executed. So the fix was to ensure the format linter is also added when a single linter is specified.

### Thing 2: Thorsten and Foreshadowing

A week ago I was on support on a glorious Monday, and then horror struck. Opsgenie decided to play the song of its people. Dotcom was unavailable ... After some confused internal screaming I logged in and started an [incident](https://app.incident.io/incidents/154). Poor Dave was also woken up by OpsGenie and was already diagnosing what was wrong. Frontend was not coming up and was stuck in pending as it was waiting for migrator to finish running since migrator is its init container. Eventually Thorsten joined and started running some queries to see why the migrator was taking so long to run. After a few tactical queries, Thorsten saw that migrator was waiting on a lock which repo-updater was holding because of a repo-sync job. He cancelled the job and then migrator was able to complete! And just like that frontend came up and dotCom was reachable again. We made sure to record the queries that were executed in the [incident](https://app.incident.io/incidents/154) summary since they were crucial in resolving the [incident](https://app.incident.io/incidents/154).

Fast forward to today and we have a similar issue in scaletestind. Only now, migrator wasn't hanging around and it went into a crashloopbackoff. Before it got restarted by kubernetes the log reported that there was a single dirty migration. @sanderginn remembered that the migrator has cli arg for this exact situation. So he patched the deployment to add this arg to the migrator invocation. By doing that we got a little forward, and now migrator was taking a long time to run ... like our previous [incident](https://app.incident.io/incidents/154)!

I remembered that we documented all the queries so I opened the [incident](https://app.incident.io/incidents/154) again while also starting up my cloud-sql-proxy with `cloud_sql_proxy -instances=sourcegraph-scaletesting:us-central1:sg-scaletesting-d2d240d290=tcp:5555` and grabbed the [incident](https://app.incident.io/incidents/154) queries from the summary we wrote. With the queries in hand I was able to identified why migrator was blocked - it was because of repo-updater and a sync job. I unblocked it by cancelling the job that was causing repo-updater to hold a lock on the table. After doing that, migrator finished up and frontend started up again!

## 2022-10-24

@jhchabran / @burmudar  quick recap about the bets and cooldown:

- Wrap up the bet and share the result
- Get stuff ready for the batch changes team
  - 200k repos
    - need to make them private
  - codehostcopy
- Extract the repo thing into its own package for resuming
- Boot up things with permissions on GHE
- Create the new bet for scale testing
- Take some time to make sure https://github.com/sourcegraph/sourcegraph/issues/42140 works.


## 2022-10-21 (occurred 2022-10-19)
@sanderginn

DevX was flagged that S2 had not been deployed to for 6 days. What turned out to be the root cause is that the [IAM bindings on the service account](https://sourcegraph.sourcegraph.com/github.com/sourcegraph/infrastructure/-/blob/gcp/org/managed_instance_folder_iam_bindings.tf?L68-76) that is in use by GitHub Actions (where CD takes place) are [frequently undone by a non-Terraform change](https://sourcegraph.slack.com/archives/C1JH2BEHZ/p1666205521693909?thread_ts=1666202912.133939&cid=C1JH2BEHZ). This causes the SA to lose the permission `compute.instances.list`, which is needed to get details on the current deployment context of the managed instance.

What made this particularly difficult to debug is that the missing permission error thrown by `gcloud` was not caught as an error by the `run.Cmd().Run()` [execution](https://sourcegraph.sourcegraph.com/github.com/sourcegraph/deploy-sourcegraph-managed@main/-/blob/util/pkg/config/config.go?L206-209&subtree=true). The function `fetchDeployment` in turn [does not return an error](https://sourcegraph.sourcegraph.com/github.com/sourcegraph/deploy-sourcegraph-managed@main/-/blob/util/pkg/config/config.go?L232&subtree=true) when there are no instances found matching the filter on line 215.
The end result is that an empty string is returned by `fetchDeployment`, which causes `ActiveDeploymentRoot` to [return an empty string too](https://sourcegraph.sourcegraph.com/github.com/sourcegraph/deploy-sourcegraph-managed@main/-/blob/util/pkg/fs/fs.go?L54-61), and ultimately leading to `ValidDockerComposeFile` to return an error when trying to [get the file status of `docker-compose`](https://sourcegraph.sourcegraph.com/github.com/sourcegraph/deploy-sourcegraph-managed@main/-/blob/util/pkg/fs/fs.go?L114), as it is not prefixed with the directory of the instance being upgraded (something like `deploy-sourcegraph-managed/sg/red`).

The difficulty of debugging is compounded by the fact that GitHub Actions are just a PITA to test. I finally just resorted to hacking the bits of code in `mi` to log/print to stdout, pushing the changes to `create-pull-request/patch` branch that was not being merged (due to the action failing). This branch is fine to throw away and is checked out by the failing job, making things a bit quicker to investigate.
The output below finally revealed what was going wrong.

```shell
2022-10-19T17:08:09.9758683Z upgrading sg to 178685
2022-10-19T17:08:10.0074999Z 👉 [1mUsing target "latest"[0m
2022-10-19T17:08:10.0076187Z 👉 [1mNo deployment found in cache, fetching[0m
2022-10-19T17:08:34.8790876Z 👉 [1mDeployment fetch result: [ERROR: (gcloud.compute.instances.list) Some requests did not succeed:  - Required 'compute.instances.list' permission for 'projects/sourcegraph-managed-sg' ][0m
2022-10-19T17:08:34.8791699Z 👉 [1mNo deployment found in cache, fetching[0m
2022-10-19T17:08:56.9425191Z 👉 [1mDeployment fetch result: [ERROR: (gcloud.compute.instances.list) Some requests did not succeed:  - Required 'compute.instances.list' permission for 'projects/sourcegraph-managed-sg' ][0m
2022-10-19T17:08:56.9437823Z 2022-10-19T17:08:56.942Z	INFO	mi/mg_upgrade.go:79	Active deployment root	{"target": "latest", "customer": "sg", "root": "", "currentDeployment": "", "customer": "sg"}
2022-10-19T17:08:56.9438630Z golden file is invalid: stat docker-compose: no such file or directory
2022-10-19T17:08:56.9443682Z ##[error]Process completed with exit code 1.
```

Security applied the Terraform config granting the appropriate permissions to the SA, after which CD resumed.

Action items:
* [monitor for CD failures](https://github.com/orgs/sourcegraph/projects/212/views/47).
* Investigate Run swallowing errored cmd

Question: is there a better way (less stone-agey approach) to debug Actions?

## 2022-10-21

@jhchabran @burmudar See https://github.com/sourcegraph/sourcegraph/issues/43282

## 2022-10-19

@jhchabran @burmudar: Due to https://sourcegraph.slack.com/archives/C02FLQDD3TQ/p1666186593157969 we looked for recent CI changes. We ruled out the request changes, but the cronjob could be a match. We reverted it in https://github.com/sourcegraph/infrastructure/pull/4097 after observing that it passed on `main` when manually applied.

## 2022-10-18

@sanderginn Beatrix had asked us over a month ago to add a missing environment variable, `VSCODE_OPENVSX_TOKEN` to the CI agents. This token [is needed](https://sourcegraph.sourcegraph.com/github.com/sourcegraph/sourcegraph/-/blob/client/vscode/scripts/publish.ts?L28) in conjunction with `VSCODE_MARKETPLACE_TOKEN` (which was set already) to run the publish job in CI.

## 2022-10-10

@jhchabran added a bot acount for the data eng team to our org. It needs admin access to the `soucegraph/analytics` repo. See [thread](https://sourcegraph.slack.com/archives/C01CSS3TC75/p1664217790860519).

Also started looking into the conn count issues mentioned by Coury over this [thread](https://sourcegraph.slack.com/archives/C032Z79NZQC/p1665434622037289). see https://github.com/sourcegraph/deploy-sourcegraph-docker/pull/865

Intervened on https://sourcegraph.slack.com/archives/C02FLQDD3TQ/p1665417493235209 as the fix seems to be stuck on GitHub. Giving it a try with a smaller repo to fix that annoying flake plaguing the e2e jobs. as described in https://github.com/sourcegraph/sourcegraph/issues/42637. PR is at https://github.com/sourcegraph/sourcegraph/pull/42816

## 2022-10-07

@burmudar the `sg_setup` action was failing on Github. Specifically the ubuntu test was failing. After fighting with bash and trying to see why it was happening, it came down to the `.bashrc` bundled with ubuntu has a piece of bash that stops the `.bashrc` execution if bash is not running in an interactive mode. So I went and made sure that when `sg` invokes a script and the shell is Bash that it adds the `-i` flag. My local and remote testing verified that this was indeed the [fix](https://github.com/sourcegraph/sourcegraph/pull/42689), but @mrnugget pointed me to an earlier [PR](https://github.com/sourcegraph/sourcegraph/pull/42031) by @sqs which basically does the inverse of what I did.

The reason @sqs did this was because on PopOS it just wasn't working and `sg setup` was being backgrounded the whole time. I've been bitten by this bug/behaviour previously before I had my Mac so I decided to dig in and try to figure out why this was happening and what was the cause. I pulled a PopOS container and Ubuntu container and set them up so that I could test the setup behaviour in each one which lead to the following "debunking" of assumptions

#### Assumption 1 - it is bash

The output `[1]+ Stopped go run ./dev/sg setup` it the same output one gets when background a script in bash when you do CTRL+Z. So naturally, this is some bashism right? I tried getting bash to output WHAT it is doing. Telling sg to invoke it with -v, -x, --debug to see what bash command leads to the backgrounding.

To my surprise, this doesn't change a thing. Nothing. You get the same output.

#### Assumption 2 - something with how we're executing stuff in sg

I thought that maybe there is something weird `sourcegraph/run` does with the way it's executing things. I wrote a small `main.go` that executes `source /root/.bashrc || true; env` using the Run and Bash methods and executed it within the PopOs container, and it worked like a charm. No issue at all.

#### Assumption 3 - This must happen in another shell to like zsh

If it happens on bash in PopOs, does it happen in zsh? Short answer, yes it does

```
zsh: suspended (tty input)  go run ./dev/sg setup
```

But this gives as at least a hint as to what is happening. What is this suspended (tty input)? This [stackoverflow](https://stackoverflow.com/questions/24056102/why-do-i-get-suspended-tty-output-in-one-terminal-but-not-in-others) post says that it could be due to a TTY setting and recommends running stty -nostop, which I did, but it didn't help much.

Digging further into this tty setting and what suspended (tty input) does. @marekweb came across the `SIGTTIN` [signal](https://www.gnu.org/software/libc/manual/html_node/Job-Control-Signals.html). To see whether it was something concrete we added to the `checks.Runner` that we should be notified of a `SIGTTIN` signal with:

```
sigs := make(os.Signal, 1)

signal.Notify(sigs, syscal.SIGTTIN)

go func() {
     sig := <- sigs
     fmt.Println("WE GOT A SIGTTIN SIGNAL")
}()
```

Running `sg` with the above code lead to at least `sg` not being backgrounded anymore, but we also didn't get our fmt message.

#### Assumption 4 - something we outputting is causing this

It was suspicous that we get all the other sg output, but as soon as the checks start running it gets backgrounded. When we made all the category checks empty, the behaviour stopped - nothing was getting backgrounded. So something being outputted causes the TTY to background the job

Maybe this helps someone else investigating this but with the above behaviour I really think its something with the TTY configuration + syscalls.

## 2022-10-05

@jhchabran and @sanderginn, GitHub outage affecting webhooks, leading to builds not triggering on Buildkite. Resolved by re-creating the webhook manually. Also added a new playbook entry https://github.com/sourcegraph/handbook/pull/5148

## 2022-09-26

@jhchabran, I swapped the GitHub QA token to use the new one, that is using an account exclusively made for that. It failed again this morning, even if on sourcegraph-bot-2. If this fails again, we'll have to move to use tokens from an OAuth app instead.

Disabled a flake https://github.com/sourcegraph/sourcegraph/issues/42062

## 2022-09-21

@jhchabran Reverted 3 PRs in a row that were blocking the main branch, see https://sourcegraph.slack.com/archives/C02FLQDD3TQ/p1663752104836109.

Quickly investigated https://github.com/sourcegraph/sourcegraph/issues/41838 and https://github.com/sourcegraph/sourcegraph/issues/39687. Ruled those out as low-prio, as they're not blocking execution. Both bug reports are ambiguous about this, so I had to asked them, just in case.

## 2022-09-05

@jhchabran Investigated in the token issues that we've seen failing the QA tests (see https://sourcegraph.slack.com/archives/C01N83PS4TU/p1662378693209219) and found out that we have been using the same token in BuildTracker. Found it out by searching in 1Password and as William was
careful enough to add an entry over there, the value matched the Buildtracker entry. We have created a new token, on the account specifically made for buildkite stuff instead, which should reduce the requests consumption on the original token.

@jhchabran Paired for a bit with Olaf, who had very weird issues with his local environment. (see https://sourcegraph.slack.com/archives/C07KZF47K/p1662390457598589). In the end, it was his `asdf` installation which was deactivated. Weirdly enough, no logs were indicating this, we only
found it empirically.

## 2022-08-24

@jhchabran This morning [INC-140] happenend and as it affected DotCom, I jumped in to help. Olaf was already there and quickly identified the problem. We rolled out a fix and I expedited its deployment. It was a frontend code issue that broke the syntax highlighting.

Olaf didn't know about `sg ci build`, so we showed it to him. Strangely though, adding his token through `sg` didn't work, so we need to double check that it's working.

## 2022-07-13

@jhchabran I've added two new how-tos, covering how to deal with custom step notifications and soft failures: https://github.com/sourcegraph/sourcegraph/pull/38718

## 2022-07-06

@jhchabran Guess how happy I was this morning to see that we went back to 13 agents and they were all stuck haha!

Two things happened. I forgot to merge the fixing PR last evening when I came back in the evening. And the other one is that apparently, and I have no idea why, but this was missing from the ROX pvc manifest:

```
M buildkite/buildkite-git-references/PersistentVolumeROX.template.yaml
@@ -18,6 +18,7 @@ spec:
     fsType: ext4
     # Create this disk with gcloud
     pdName: buildkite-git-references-$BUILDKITE_BUILD_NUMBER # interpolate this value
+    readOnly: true
   capacity:
     storage: 16G
   persistentVolumeReclaimPolicy: Delete
```

Once this has been applied, agents started working again, being able to concurrently mount the git ref PVC. Phew.

## 2022-07-05

@jhchabran Valery reported that some agents are not firing up, leading to build being stuck. A quick check showed that only 31 agents are available.

Many agents are stuck in "container creating" or "pending" state, for about 3h with the following log message:

```
stream logs failed container "dind" in pod "buildkite-agent-stateless-3d11562a88f44d6c83f91422a358bad3cksqj" is waiting to start: ContainerCreating for buildkite/buildkite-agent-stateless-3d11562a88f44d6c83f91422a358bad3cksqj (dind)                                                                                                                 stream logs failed container "buildkite-agent-stateless" in pod "buildkite-agent-stateless-3d11562a88f44d6c83f91422a358bad3cksqj" is waiting to start: ContainerCreating for buildkite/buildkite-agent-stateless-3d11562a88f44d6c83f91422a358bad3cksqj (buildkite-agent-stateless)
```

```
Normal   NotTriggerScaleUp  3m12s                cluster-autoscaler  pod didn't trigger scale-up: 6 node(s) had volume node affinity conflict, 2 in backoff after failed scale-up

Warning  FailedScheduling   64s (x9 over 3m25s)  default-scheduler   0/133 nodes are available: 1 node(s) had taint {node.kubernetes.io/disk-pressure: }, that the pod didn't tolerate, 130 Insufficient cpu, 2 Insufficient memory, 2 node(s) had volume node affinity conflict.

Warning  FailedScheduling   6s (x7 over 2m35s)   default-scheduler   0/133 nodes are available: 129 Insufficient cpu, 2 Insufficient memory, 2 node(s) had taint {node.kubernetes.io/disk-pressure: }, that the pod didn't tolerate, 2 node(s) had volume node affinity conflict.
```

After investigating further with Sanders, we noticed that a GKE updated was automatically applied. We went from 1.21 to 1.22, which had the unexpected side effect of breaking the `buildkite-git-references` volumes. This resulted in agents being stuck in `pending` or `containerCreating` and [exhausted all the disk quota](https://console.cloud.google.com/iam-admin/quotas?referrer=search&project=sourcegraph-ci).

```
Node scale up in zones us-central1-c associated with this pod failed: GCE quota exceeded. Pod is at risk of not being scheduled
```

After killing all jobs to release the resources, we scaled down the total agent count, because the current size for the agent base disks times 250 is greater than the allocated quota.

Still, we a big problem remained, even though we were able to get agent scheduled, they were still stuck as they were failing to mount the `buildkite-git-references` volume.

We tried creating a new one manually, which didn't fix the problem. After digging further, noticing that only a single agent at a time could mount the volume, whereas it's expected that it can be mounted by many, we started investigating how the upgrade could have affected this.

The `buildkite-git-references` volumes have two access modes, `RWO` and `ROX`. The former, being used by the populating job and the latter by the agents themselves. Digging in [the docs showed us](https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/readonlymany-disks#volume-snapshot) that this not how it's supposed to be done.

Following the above guide didn't work, as we kept getting `multiattach` errors on containers creation. We wrongly assumed that the [persistent disk CSI driver](https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/gce-pd-csi-driver) was [already enabled as our TF indicates](https://sourcegraph.com/github.com/sourcegraph/infrastructure/-/blob/buildkite/gke.tf?L38), which wasn't the case.

Fixing this, we still could not get the approach with a PV/PVC for the `RWO` access and one for `ROX` to work with the `datasource` approach recommend by the docs. We instead found a [SO post](https://stackoverflow.com/questions/64128047/migrating-rom-rwo-persistent-volume-claims-from-in-tree-plugin-to-csi-in-gke) which gave a more manual approach that worked, see [the final PR](https://github.com/sourcegraph/infrastructure/pull/3616).

In retrospective, the next time we see a k8s update, we should instantly TF apply. But eh, that was the first time I've seen this so, lesson learned the hard way. Will take some time later to devise follow-up actions.

## 2022-07-01

@jhchabran

A [test opsgenie alert](https://opsg.in/a/i/sourcegraph/3c603aeb-a223-4b41-a4a4-94d7467dba7d-1656692903240) was sent (don't know from where it comes from). OpsGenie integration was sent over #dev-experience which is kinda noisy and not the right place. I moved it to #dev-experience-internal instead.

## 2022-06-30

@jhchabran

The decision with Nate has been taken, it-tech-ops will deal with GitHub membership requests for the time being, but will stay out of the Github-owners handle.

## 2022-06-23

@jhchabran Wired Dogfood with its Sentry project, as asked by Camden. I capped the projet to 30 errors per hour to avoid blowing away the quotas.

## 2022-06-22

@jhchabran

Valery reported that there is some CI infra issue with `Gulp`. Investigated and filed https://github.com/sourcegraph/sourcegraph/issues/37548 which should be a really quick fix. Ping the relevant teams and individuals, so they can fix it on their own.

---

The `Directory renamed before its status could be extracted` is [back again](https://buildkite.com/sourcegraph/sourcegraph/builds/155788#018187cb-9d4e-4a86-8785-50ff844a05e4/153-157). A Grafana search in the logs with the query `{app="buildkite"} |= "Directory renamed before its status could be extracted"` shows that it started happening again yesterday, but hasn't happened yet today.

```sourcegraph
context:@sourcegraph/all repo:^github\.com/sourcegraph/sourcegraph$ file:^enterprise/dev/ci/internal/buildkite/cache\.go cachePluginName  patternType:literal
```

Shows that we're still using the fork with `bsdtar`, so again, no explanation yet about why happened.

---

[Failure on client jobs involving the cache](https://sourcegraph.slack.com/archives/C02FLQDD3TQ/p1655890023860649): https://buildkite.com/sourcegraph/sourcegraph/builds/155910#01818aa0-ddfb-4aa4-a5f4-ba7ae11a2a7f/137-150

```
>[node_modules] - 🔥 Cache hit: s3://sourcegraph_buildkite_cache/sourcegraph/sourcegraph/cache-node_modules-3f21cee23df6eba8d19bc1da7c6176d16f8a859e.tar
download failed: s3://sourcegraph_buildkite_cache/sourcegraph/sourcegraph/cache-node_modules-3f21cee23df6eba8d19bc1da7c6176d16f8a859e.tar to ./cache-node_modules-3f21cee23df6eba8d19bc1da7c6176d16f8a859e.tar An error occurred (InvalidRange) when calling the GetObject operation: The requested range cannot be satisfied.
```

Quick Google search showed that `InvalidRange` is an API error on GCP end.  As for the `MODULE_NOT_FOUND`, no idea so far what caused it. But its proximity with the api errors makes it quite suspect.

As it only happened [3 times, and only today](https://bit.ly/3NkjR8Y) let's just keep monitoring this for now.

## 2022-06-20

@bobheadxi

A bunch of things happened:

1. GitHub webhooks go down, causing builds to not run
2. Dispatcher fetches of Buildkite metrics [receives an invalid response](https://github.com/sourcegraph/infrastructure/pull/3569#discussion_r902035694), possibly caused by the above, that causes a lack of agents for several hours
   1. Undetected! I added [a new alert rule on total agent count](https://console.cloud.google.com/monitoring/alerting/policies/7627077487159874251?folder=true&organizationId=true&project=sourcegraph-ci) to try and catch this in the future (we currently only alert on "expected dispatch duration", which requires the dispatcher to be online)
   2. [Using our logging + sentry integration](https://github.com/sourcegraph/sourcegraph/issues/36923) might help in the future as well by reporting errors (but we need to catch panics as well)
3. I roll back the dispatcher change, but now agents are struggling to start up. We are hitting CPU quotas, but Erik notices [we are actually hitting storage quotas](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1655767262475429?thread_ts=1655763780.493669&cid=C01N83PS4TU) (over 100TB!) ([quotas page](https://console.cloud.google.com/iam-admin/quotas?referrer=search&folder=true&organizationId=true&project=sourcegraph-ci))

A lot of these disks are mysterious `pvc-${uuid}` disks of various sizes, up to 200GB each, that don't seem to be in use by anything. They seem to be coming from things deployed to the default `ns-sourcegraph` namespace, as indicated by the `Description` on some of the disks, e.g. `pvc-5096b7b6-807e-4392-8f68-8305bf1c0a59`:

```json
{"kubernetes.io/created-for/pv/name":"pvc-5096b7b6-807e-4392-8f68-8305bf1c0a59","kubernetes.io/created-for/pvc/name":"data-indexed-search-0","kubernetes.io/created-for/pvc/namespace":"ns-sourcegraph","storage.gke.io/created-by":"pd.csi.storage.gke.io"}
```

I wrote a quick script to dump these as an interim measure (based on [`prune-pvcs.sh`](https://sourcegraph.com/github.com/sourcegraph/infrastructure/-/blob/buildkite/buildkite-git-references/prune-pvcs.sh)) - this deletes 550 disks, bringing us down to 20TB of disk usage:

```sh
gcloud compute disks list --project sourcegraph-ci --filter='description:ns-sourcegraph AND NOT users:*' --format='value(name)' |
  while read -r disk; do gcloud compute disks delete ${disk} --project sourcegraph-ci --zone us-central1-a --quiet ; done
```

I'm pretty sure these disks are coming from `deploy-sourcegraph`, namely:

https://sourcegraph.com/github.com/sourcegraph/deploy-sourcegraph@683e455055d753a684302cdb98783cb255316709/-/blob/tests/integration/restricted/test.sh?L47

In `sourcegraph/sourcegraph`, we create a more descriptive namespace:

https://sourcegraph.com/github.com/sourcegraph/sourcegraph@3913ffdfb5acfac8319f59c556b54f05a0f4ca27/-/blob/dev/ci/integration/cluster/test.sh?L12:72=

This seems to have happened before: https://github.com/sourcegraph/deploy-sourcegraph/pull/4086, and the created disks are after the introduction of this change.

However, mysteriously we see that the script seems to be working as expected ([example](https://buildkite.com/sourcegraph/deploy-sourcegraph/builds/26065#01818388-3009-4394-9217-1c5969102d6f/235-489)):

```none
+ CLEANUP='kill 11345; rm -rf generated-cluster; kubectl delete namespace ns-sourcegraph --timeout=180s; gcloud container clusters delete ds-test-restricted-01818362 --zone us-central1-a --project sourcegraph-ci --quiet; '
+ kubectl -n ns-sourcegraph port-forward svc/sourcegraph-frontend 30080
+ sleep 2
+ curl --retry-connrefused --retry 2 --retry-delay 10 -m 30 http://localhost:30080
curl: (7) Failed to connect to localhost port 30080: Connection refused
+ bash -c 'kill 11345; rm -rf generated-cluster; kubectl delete namespace ns-sourcegraph --timeout=180s; gcloud container clusters delete ds-test-restricted-01818362 --zone us-central1-a --project sourcegraph-ci --quiet; '
namespace "ns-sourcegraph" deleted
```

Notably, we see that `CLEANUP` is correctly set to what seems to be a reasonable cleanup process, and the namespace gtets appropriately deleted. This should remove any PVCs, which should prompt GKE to delete the associated disks - this is what we do in `sourcegraph-sourcegraph` as well, and its addition is described in https://github.com/sourcegraph/deploy-sourcegraph/pull/4086:

https://sourcegraph.com/github.com/sourcegraph/sourcegraph@3913ffdfb5acfac8319f59c556b54f05a0f4ca27/-/blob/dev/ci/integration/cluster/test.sh?L27:1-31:2=

However, there we *don't* delete the cluster with `gcloud container clusters delete`. [In the docs](https://cloud.google.com/sdk/gcloud/reference/container/clusters/delete) it's noted (emphasis mine):

> GKE will attempt to delete the following resources. **Deletion of these resources is not always guaranteed**:
>
> - External load balancers created by the cluster
> - Internal load balancers created by the cluster
> - **Persistent disk volumes**

Possible next steps:

- Maybe we shouldn't be explicitly delete the cluster at all? Though cluster creation does seem necessary here, so it looks like this is necessary.
- Use a more descriptive namespace: https://github.com/sourcegraph/deploy-sourcegraph/pull/4144
- Can we get rid of these tests entirely?

@DURATION=3h

## 2022-06-16

@jhchabran Following up on the request from Security that @bobheadxi mentioned in yesterday's entry, I gave a spin out of curiosity to simply patching the package itself in the Alpine repository. I was pleased to see that it was really straightforward.

```sh
docker run -it --name alpine-sdk alpine:3.12.12
apk add alpine-sdk
adduser tech
addgroup tech wheel
addgroup tech abuild
su tech
git clone https://gitlab.alpinelinux.org/alpine/aports
abuild-keygen -a -i
cd aports/main/postgresql
# edited the APKBUILD and bumped the version
abuild checksum
make
abuild -r
# tarballed ~/packages
```

```sh
docker run -it --name alpine-test-the-build alpine:3.12.12
# get the tarball under ~
apk add --allow-untrusted --repository ~/packages/main/ postgresql=12.11-r0
# ... started the pg server and made sure it's running the right version
```

I gave security the build, asked them what's the priority on this, I'll probably have to dig in further to understand what's the process of updating a package in the main repo in an old branch such as `v3.12.12` in Alpine.

@DURATION=45m

## 2022-06-15

@bobheadxi

Security is looking for help to upgrade to Postgres 12.11 in the single-container image: https://github.com/sourcegraph/sourcegraph/pull/37227
However, 3.12 is the only branch with postgres 12, and it stops at 12.10: https://pkgs.alpinelinux.org/packages?name=postgresql&branch=v3.12, while 3.13 jumps to postgres 13.7: https://pkgs.alpinelinux.org/packages?name=postgresql&branch=v3.13

In `v3.15/community`, there is https://pkgs.alpinelinux.org/package/v3.15/community/aarch64/postgresql12 which seems to be the only package that satisfies the requirements, so I try to use it:

```dockerfile
RUN apk add --no-cache --upgrade --verbose \
    --repository=http://dl-cdn.alpinelinux.org/alpine/v3.15/community \
    'postgresql12>=12.11' \
    'postgresql12-contrib>=12.11'
```

We get this error:

```none
#6 2.325   postgresql-common (no such package):
#6 2.325     required by: postgresql12-12.11-r0[postgresql-common]
#6 2.325                  postgresql12-client-12.11-r0[postgresql-common]
#6 2.325   so:libicui18n.so.69 (no such package):
#6 2.325     required by: postgresql12-12.11-r0[so:libicui18n.so.69]
#6 2.325   so:libicuuc.so.69 (no such package):
#6 2.325     required by: postgresql12-12.11-r0[so:libicuuc.so.69]
#6 2.325   so:libldap.so.2 (no such package):
#6 2.325     required by: postgresql12-12.11-r0[so:libldap.so.2]
```

Using [this reference](https://alpine.pkgs.org/3.15/alpine-community-x86_64/postgresql12-12.11-r0.apk.html) to try and piece together what's missing, I ended up with something like this:

```dockerfile
FROM sourcegraph/alpine-3.14:154143_2022-06-13_1eababf8817e@sha256:f1c4ac9ca1a36257c1eb699d0acf489d83dd86e067b1fc3ea4a563231a047e05
USER root

# postgres12 requires these things that aren't in /community
RUN apk add --no-cache --verbose \
    --repository=http://dl-cdn.alpinelinux.org/alpine/v3.15/main \
    'icu-libs' \
    'postgresql-common'
# 3.15 has libldap 2.6 which cannot be used by postgres12, so we try to get it from 3.14
RUN apk add --no-cache --verbose \
    --repository=http://dl-cdn.alpinelinux.org/alpine/v3.14/main \
    'libldap'

RUN apk add --no-cache --upgrade --verbose \
    --repository=http://dl-cdn.alpinelinux.org/alpine/v3.15/community \
    'postgresql12>=12.11' \
    'postgresql12-contrib>=12.11'
```

Sadly, even this results in unsatisfiable dependencies:

```none
#7 2.866 ERROR: unable to select packages:
#7 2.925   so:libldap.so.2 (no such package):
#7 2.925     required by: postgresql12-12.11-r0[so:libldap.so.2]
```

No cigar. Thinking about using:

```dockerfile
COPY --from=$(postgres image)
```

But postgresql isn't really just a single binary we can copy it seems. We could use `FROM index.docker.io/sourcegraph/postgres-12-alpine` as a base, but then we'd have a strict dependency between the postgres image and server, i.e. `server@abcde` requires `postgres-12-alpine@abcde` to build, but we don't have a way to ensure one is built before the other.
This will probably work but I'm not sure it's the right way to go.

More discussion, thought not too much new information from the above at time of writing: https://sourcegraph.slack.com/archives/C02FLQDD3TQ/p1655316909723799?thread_ts=1655215416.930249&cid=C02FLQDD3TQ

@DURATION=3h

## 2022-06-13

@jhchabran We saw a [strange failure](https://buildkite.com/sourcegraph/sourcegraph/builds/154191#01815d66-41f0-46a8-9ebf-10e4806cba3a/114-142}) while building Docker images:

```
failed to solve with frontend dockerfile.v0: failed to create LLB definition: failed to do request: Head https://private-docker-registry:5000/v2/library/postgres/manifests/sha256:b815f145ef6311e24e4bc4d165dad61b2d8e4587c96cea2944297419c5c93054?ns=docker.io: http: server gave HTTP response to HTTPS client
```

It caused buildchecker to kick in and lock the `main` branch. Restarting the `private-docker-registry` fixed the problem.

## 2022-06-09

@bobheadxi After repeated attempts to cut a patch release for 3.39, there were still build issues where compilation errors were appearing on Go 1.17-incompatible code (the `any` type alias) despite no such code being present in Coury's 3.39 branch (`3.39.1-insightsdb-patch`). Eventually pinned this down to the `src-cli` docs generation code pulling down the *latest* version of `src-cli` to generate docs, regardless of what version of the branch we are on:

https://sourcegraph.com/github.com/sourcegraph/sourcegraph@3.39.1-insightsdb-patch/-/blob/doc/cli/references/doc.go?L101

I recommended the following patch after confirming the command worked locally ([08090219ed7](https://sourcegraph.com/github.com/sourcegraph/sourcegraph/-/commit/08090219ed7f09892ff63033c0f80899ea95e17b)):

```diff
- goGet := exec.Command("go", "get", "github.com/sourcegraph/src-cli/cmd/src")
+ goGet := exec.Command("go", "get", "github.com/sourcegraph/src-cli/cmd/src@3.39.2")
```

I'm very impressed this hasn't caused issues in the past, e.g. by being caught by the `go generate` check during lints, though I suppose the `src-cli` generated docs output might not change that often. This patch currently only exists on `3.39.1-insightsdb-patch` - we might want to include some variation of it in `main`, but I do not think it'll be very ergonomic to update (i.e. it'll require some automated PR to `sourcegraph` whenever `src-cli` is updated).

## 2022-06-08

@jhchabran and @william The executors image, was failing consistently with various errors, with no direct changes on its code, and was still working yesterday. Those were caused by an outage on Launchpad which broke adding gitcore ppa. After a while, those errors stopped appearing, but we still saw a very obscure failure. The executor job is building images for both GCP and AWS, but the GCP one was failing while building a Docker image, with no apparent reason. A silent stop when running apt-get update. It took us a while to find out that what were seeing here was a kernel panic, which was [reported earlier this morning](https://bugs.launchpad.net/ubuntu/+source/linux-aws/+bug/1977919) on the ubuntu 20.04 LTS daily build that is being used to run the VM building the GCP image. The solution was to [pin down the image](https://github.com/sourcegraph/sourcegraph/pull/36782) to the previous image, that is known to work.

It was hard to see, because the [output from the script](https://buildkite.com/sourcegraph/sourcegraph/builds/152871#0181425c-2a81-48bd-b06b-e2b61795a9a6/363-1180) was just `+ apt-get update` then `gcp: build has errors` and goodbye. It took look at the logs on the serial port to finally spot that it was a [kernel panic](https://console.cloud.google.com/compute/instancesDetail/zones/us-central1-c/instances/packer-62a078b1-78e2-2f65-3aae-62c470946e69/console?port=1&authuser=1&project=sourcegraph-ci).

Along the way we found out that the executors job was run on every build where it shouldn't have been. After a bit of digging, it came up that the method used to compute if a new executor build is required or not wasn't stable anymore. William brilliantly rememebered reading about a new flag in Go's changelog recently, which led to [a fix](https://github.com/sourcegraph/sourcegraph/pull/36778).

## 2022-06-07

@jhchabran This morning, we started seeing linter errors such as https://buildkite.com/sourcegraph/sourcegraph/builds/152538#01813cb1-4def-4b96-9ad5-f7a2fcc39403. We did not manage to reproduce those errors locally, neither on our laptops or within linux VMs. Heck, even on CI, when running the linter it passed. This was the hint that we're seeing a race in between the Go generators and the docsite checker. After trying a few hacks, I simply disabled the docsite linter until we can fix the whole issue.

## 2022-06-02

@jhchabran Noah stumbled on an edge case where we're trying to use `os.Rename` across different partitions which always fail. Fix in https://github.com/sourcegraph/sourcegraph/pull/36469 DURATION=10m

## 2022-06-01

@jhchabran Jean Du Plessis gave me admin privileges, so I was able to fix the pr-auditor-check issue on sg/infra. DURATION=15m Thanks so much to the person that added the env var in 1Password, it enabled to be sure I was targeting the right account.

## 2022-05-30

@jhchabran

Paired with Oleg and Jason to help them kickstart https://github.com/sourcegraph/sourcegraph/issues/30536#issuecomment-1032641034. We briefly went over how the CI pipeline works, then rapidly iterated through how to upload something in GCP, create a simple script to do so, where to insert their client bundle upload in the pipeline and more importantly how to make the feedback loop as short as possible:

```sourcegraph
context:global repo:^github\.com/sourcegraph/sourcegraph$@b530444 file:^enterprise/dev/ci/internal/ci/pipeline\.go if c.Branch == "fpdx/upload-fe-bundle" {...} patternType:structural
```

They're going to iterate on that and will come back to us for the reviewing the final PR.

## 2022-05-24

@jhchabran

[The same exact thing happend to Joe Chen](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1653375898615149), but this time we noticed very quickly that a CI ref file has not been updated and committing https://github.com/sourcegraph/sourcegraph/commit/a9b15bdc32409bbec9c5522c55d4b1f8b20c299c fixed the issue. As for why we're seeing this pattern of behaviour where `sg lint go` hangs if the `sg generate` commands results in a dirty repo, we don't know yet. What's for sure though is that the buffering of the ouptut which makes it appear only once the command has exited is really unpractical when it comes to debugging, and we need to fix this asap.  DURATION=30m .

@jhchabran

[Thorsten noticed that the `main` branch is broken due to the 3.40.0 release](https://sourcegraph.slack.com/archives/C032Z79NZQC/p1653399008176829), it went unnoticed because the support handle for the team is set this week on Robert, which at this time is sleeping. After some investigation, we noticed that Alex Ostrikov merged a commit 6 hours ago that introduced a backward incompatible change, albeit a peculiar one: the database change itself is backward compatible, but not the tests, which were accounting for faulty behaviour. The fix was to [port the flake files to the new 3.40.0 version](https://github.com/sourcegraph/sourcegraph/pull/35942) and [we added along the way a word of warning about this edge case](https://github.com/sourcegraph/sourcegraph/pull/35945). DURATION=45m

@jhchabran

[An arm64 image was used as a base for CAdvisor](https://sourcegraph.slack.com/archives/CMBA8F926/p1653411739808069) and broke the main branch. [Reverted the PR](https://github.com/sourcegraph/sourcegraph/pull/35959) and advised the user on how to fix this. Also, the CI pipeline generator isn't detecting changes in the images and their PR did not catch this. DURATTION=20m

## 2022-05-23

@jhchabran

[Eric Fritz had trouble with a PR which added new go generate statements](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1653322924295219) We paired with William on this and found out that the result are inconsistent in local, `sg generate` completes, but sometimes `sg lint go` hangs. On CI `sg lint go` consistently hangs. As it was pretty late for us, we unblocked Eric by giving him a branch with a patched version that outputted things straight to stdout. Strangely, forcing the `sg generate` code to produce output did work consistently, which hints at some buffering issue. DURATION=60m.

## 2022-05-18

@jhchabran

[Valery mentioned that GitStart is blocked on their PR](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1652866114007209) with issuse related to node versions. The problem was rather simple, their pipeline was simply not calling `asdf` at all. As we recently rolled out `asdf` installation into a plugin, that was a rather straightforward fix: https://github.com/sourcegraph/eslint-config/pull/249 DURATION=30m

## 2022-05-12

@jhchabran

Investigated [the issue the failing back compat tests](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1652311475381819?thread_ts=1652294119.054949&cid=C01N83PS4TU) and found out that there is an [where the go asdf plugin](https://github.com/kennyp/asdf-golang) reverts to Go 1.18.1 for some obscure reason.

I first reproduced the error on Buildkite to establish that it's not transient somehow and managed to reproduce it locally by running the back-compat/test.sh script manually with the same arguments that are present in the failing step. I had to tweak things a bit to make it work locally, notably adjusting the `find` command, disabling asdf installation part entirely to circumvent issues with some tools not being available for `arm64`. From there, after triple checking that the issue was indeed coming from the `go test` command, I started to bisect the package list to understand which one was causing the error. After a few tries, I managed to get a zero exit code by excluding the `lib/codeintel/tools` folder and finally isolated `lib/codeintel/tools/lsif-repl` to be the culprit.

```
# This will show exit code 2
./dev/ci/go-backcompat/test.sh exclude github.com/sourcegraph/sourcegraph/enterprise/internal/codeintel/stores/dbstore github.com/sourcegraph/sourcegraph/enterprise/internal/codeintel/stores/lsifstore github.com/sourcegraph/sourcegraph/enterprise/internal/insights github.com/sourcegraph/sourcegraph/internal/database github.com/sourcegraph/sourcegraph/internal/repos github.com/sourcegraph/sourcegraph/enterprise/internal/batches github.com/sourcegraph/sourcegraph/cmd/frontend github.com/sourcegraph/sourcegraph/enterprise/internal/database github.com/sourcegraph/sourcegraph/enterprise/cmd/frontend/internal/batches/resolvers


# This will show exit code 0 (noted the final excluded package)
./dev/ci/go-backcompat/test.sh exclude github.com/sourcegraph/sourcegraph/enterprise/internal/codeintel/stores/dbstore github.com/sourcegraph/sourcegraph/enterprise/internal/codeintel/stores/lsifstore github.com/sourcegraph/sourcegraph/enterprise/internal/insights github.com/sourcegraph/sourcegraph/internal/database github.com/sourcegraph/sourcegraph/internal/repos github.com/sourcegraph/sourcegraph/enterprise/internal/batches github.com/sourcegraph/sourcegraph/cmd/frontend github.com/sourcegraph/sourcegraph/enterprise/internal/database github.com/sourcegraph/sourcegraph/enterprise/cmd/frontend/internal/batches/resolvers github.com/sourcegraph/sourcegraph/lib/codeintel/tools/lsif-repl

# Go back to the branch, resetting the local mess
git reset --hard HEAD && rm -Rf migrations && git co . && git co main-dry-run/debug-back-compat
```

Surprisingly, this package has no test and is just a `main.go`. Running a `git blame` on the file showed that the last change there was a fix from Keegan to make it work with Go `1.18.1`. But the back compat tests are supposed to be running Go `1.17.5`. Running `go version` within the `lsif-repl` folder showed for some reason, even if the `/.tool-versions` specifies Go `1.17.5`, it's actually Go `1.18.1` that is currently running.

This put me on track toward an `asdf` related issue, a few prints in `asdf` code confirmed it. There is an [issue](https://github.com/kennyp/asdf-golang/issues/79) about someone reporting a similar issue. Reading the doc showed that there is some questionable behaviour if the `legacy_version` setting is enable. Disabling it in my `~/.asdfrc` solved the issue and now Go `1.17.1` is the current version, as expected, within the `lsif-repl` folder.

Ran a quickbuild over https://buildkite.com/sourcegraph/sourcegraph/builds/147181 to confirm that we can just drop the `.nvmrc` file and opened a PR with a new agent image that has that setting disabled.

---

Failures were reported on the deploy-sourcegraph-cloud pipeline, due to `asdf` installation of `helm` failing. It was caused by an HTTP timeout. I submitted two PRs to fix these on the plugin themselves over https://github.com/Antiarchitect/asdf-helm/pull/12 and https://github.com/luizm/asdf-shfmt/pull/7.

## 2022-05-11

@jhchabran

A [PR broke the main branch](https://sourcegraph.slack.com/archives/C07KZF47K/p1652270071646589?thread_ts=1652268753.495069&cid=C07KZF47K) and I had to revert it. I asked the author if he knew what to do in case of being mentioned by Buildchecker, just out of curiosity and he did. That person did not know about the other run types btw (and he's been with us since Nov '21).

I submitted a quick PR along the way for the wording in Buildchecker, trying to foster more autonomy from our users: https://github.com/sourcegraph/sourcegraph/pull/35291

@bobheadxi @davejrt

Major outage with asdf caching `tar` failures. Even with that [fixed](https://github.com/sourcegraph/sourcegraph/pull/35334/commits/232fd75c46a32f3a0e0d12633266190de9b7ee02), [back-compat tests continued to fail](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1652308349109149?thread_ts=1652294119.054949&cid=C01N83PS4TU). Eventually [disabled back-compat tests](https://github.com/sourcegraph/sourcegraph/pull/35334#issuecomment-1124362668). $DURATION=90m

## 2022-05-04

@bobheadxi

Made `pr-auditor` required in `sourcegraph/sourcegraph` on [request](https://sourcegraph.slack.com/archives/C01G34KFYE4/p1651677045477279?thread_ts=1651665256.667259&cid=C01G34KFYE4). Noticed issue where `pr-auditor` checks do not persist, and PRs get stuck on "waiting for status" - we [can't set status by branch](https://github.com/sourcegraph/sourcegraph/pull/34913#issuecomment-1117563565), so we need to register the action to run on the `synchronize` event ([#34914](https://github.com/sourcegraph/sourcegraph/pull/34914)) i.e. when a push happens as well. We'll need to propagate this fix to all repos with a batch change if more repos make this a required check. $DURATION=15m

Hadolint script broke. The shell script was a bit cryptic so I just moved the whole thing into `sg`: [#34926](https://github.com/sourcegraph/sourcegraph/pull/34926) $DURATION=30m

A few teammates reported occasionally running into *kernel* panics with `sg start`. Consensus seems to be that it's probably not directly related to `sg`, but might be worth looking into - for now it appears infrequent and I can't make much sense of it. [Thread](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1651696165060899), [core dump](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1651699647925089?thread_ts=1651696165.060899&cid=C01N83PS4TU) $DURATION=15m

Worked on a dependency version guard for an incident a few weeks ago: [#34929](https://github.com/sourcegraph/sourcegraph/pull/34929) $DURATION=30m

## 2022-05-02

@jhchabran Noticed that the builds were stuck this morning, because a new version of rust protobuf-codegen has been released and the dev/generate.sh script wasn't setting any expectation about which version to get. As the new release was publish, all our builds started to fail. I have put up a PR https://github.com/sourcegraph/sourcegraph/pull/34756 with the fix. $DURATION=20m

## 2022-04-28

@bobheadxi

Spent a lot of time trying to make firewall popups go away. Pursued the firewall command approach for a long time (https://github.com/sourcegraph/sourcegraph/pull/34680), before giving up and finding an infinitely easier solution: https://github.com/sourcegraph/sourcegraph/pull/34714 $DURATION=240m

## 2022-04-26

@jhchabran Another round of issues around the firewall, this time after the merge or @bobheadxi's fix. After Zooming with the person that asked for help, it became apparent that some services weren't caught by the code, which I patched here https://github.com/sourcegraph/sourcegraph/pull/34501.  $DURATION=45m

## 2022-04-22

@jhchabran Observed two builds (here and here) failing due jobs losing their agents:

```
Node condition FrequentContainerdRestart is now: False, reason: NoFrequentContainerdRestart 	NoFrequentContainerdRestart 	Apr 22, 2022, 5:50:28 PM 	Apr 22, 2022, 5:50:28 PM 	1
Node condition FrequentDockerRestart is now: False, reason: NoFrequentDockerRestart 	NoFrequentDockerRestart 	Apr 22, 2022, 5:50:27 PM 	Apr 22, 2022, 5:50:27 PM 	1
Node condition FrequentKubeletRestart is now: False, reason: NoFrequentKubeletRestart 	NoFrequentKubeletRestart 	Apr 22, 2022, 5:50:26 PM 	Apr 22, 2022, 5:50:26 PM 	1
Node condition FrequentUnregisterNetDevice is now: False, reason: NoFrequentUnregisterNetDevice 	NoFrequentUnregisterNetDevice 	Apr 22, 2022, 5:50:26 PM 	Apr 22, 2022, 5:50:26 PM 	1
Node gke-default-buildkite-main-ecd9ee7d-s9r6 status is now: NodeHasNoDiskPressure 	NodeHasNoDiskPressure 	Apr 22, 2022, 5:49:49 PM 	Apr 22, 2022, 5:50:19 PM 	7
Node gke-default-buildkite-main-ecd9ee7d-s9r6 status is now: NodeNotReady 	NodeNotReady 	Apr 22, 2022, 5:50:19 PM 	Apr 22, 2022, 5:50:19 PM 	1
Node gke-default-buildkite-main-ecd9ee7d-s9r6 status is now: NodeHasSufficientMemory 	NodeHasSufficientMemory 	Apr 22, 2022, 5:49:49 PM 	Apr 22, 2022, 5:50:19 PM 	7
Node gke-default-buildkite-main-ecd9ee7d-s9r6 status is now: NodeHasSufficientPID 	NodeHasSufficientPID 	Apr 22, 2022, 5:49:49 PM 	Apr 22, 2022, 5:50:19 PM 	7
Started Kubernetes kubelet. 	KubeletStart 	Apr 22, 2022, 4:55:25 PM 	Apr 22, 2022, 5:49:49 PM 	2
invalid capacity 0 on image filesystem 	InvalidDiskCapacity 	Apr 22, 2022, 5:49:49 PM 	Apr 22, 2022, 5:49:49 PM 	1
Updated Node Allocatable limit across pods 	NodeAllocatableEnforced 	Apr 22, 2022, 5:49:49 PM 	Apr 22, 2022, 5:49:49 PM 	1
Starting kubelet. 	Starting 	Apr 22, 2022, 5:49:49 PM 	Apr 22, 2022, 5:49:49 PM 	1
Starting Docker Application Container Engine... 	DockerStart 	Apr 22, 2022, 4:55:25 PM 	Apr 22, 2022, 5:49:39 PM 	4
Node condition FrequentContainerdRestart is now: Unknown, reason: NoFrequentContainerdRestart 	NoFrequentContainerdRestart 	Apr 22, 2022, 5:48:25 PM 	Apr 22, 2022, 5:48:25 PM 	1
Node condition FrequentDockerRestart is now: Unknown, reason: NoFrequentDockerRestart 	NoFrequentDockerRestart 	Apr 22, 2022, 5:47:25 PM 	Apr 22, 2022, 5:47:25 PM 	1
Node condition FrequentKubeletRestart is now: Unknown, reason: NoFrequentKubeletRestart 	NoFrequentKubeletRestart 	Apr 22, 2022, 5:46:25 PM 	Apr 22, 2022, 5:46:25 PM 	1
Node condition FrequentUnregisterNetDevice is now: Unknown, reason: NoFrequentUnregisterNetDevice 	NoFrequentUnregisterNetDevice 	Apr 22, 2022, 5:46:25 PM 	Apr 22, 2022, 5:46:25 PM 	1
Node gke-default-buildkite-main-ecd9ee7d-s9r6 status is now: NodeNotReady 	NodeNotReady 	Apr 22, 2022, 5:42:59 PM 	Apr 22, 2022, 5:42:59 PM 	1
```

Reading through the Buildkite docs, we can specifically address this case and force a retry: https://buildkite.com/docs/pipelines/command-step#automatic-retry-attributes, by watching for the -1 exit status: https://github.com/sourcegraph/sourcegraph/pull/34370

@jhchabran Depguard has [failed again, off a fresh branch from main](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1650638888337509), so I had to disable it again.

## 2022-04-19

@jhchabran

Got asked to [help with a CD issue](https://sourcegraph.slack.com/archives/CMBA8F926/p1650372090576579), Renovate had not picked a new PR in 8h. We uncovered weird behaviour where Renovate was updating the same PR over and over, but had a failed check dating from when we removed `fd` from being installed in every agent. This had been fixed, but the PR having been opened for a month, it was still there. Closing it apparently got Renovate to open a new PR. Coincidentally, Cloudflare had issues in Germany has well, and Renovate seems to be hosted in Germany too. We have no clue on what the exact problem. $DURATION=45m

## 2022-04-13

@davejrt

`Sourcegraph Cluster (deploy-sourcegraph) QA` tests failing on main with [this build](https://buildkite.com/sourcegraph/sourcegraph/builds/142212#4f3be801-87b7-4f74-baae-e68b54d3fc14/366-368). Some investigation showed [this commit](https://k8s.sgdev.org/github.com/sourcegraph/deploy-sourcegraph/-/commit/f0d13887f6a3f98bb4f518c29ce1d9afc37fc093?visible=3) broke the `low-resource` overlay. I fixed the the `low-resource` overlay with [this commit](https://k8s.sgdev.org/github.com/sourcegraph/deploy-sourcegraph/-/commit/0a8d240f889a8b0b2b0f4ee84fb63e7f21eced82?visible=3) and a [follow up commit](https://k8s.sgdev.org/github.com/sourcegraph/deploy-sourcegraph/-/commit/0a8d240f889a8b0b2b0f4ee84fb63e7f21eced82?visible=3) that adds a test to ensure that all overlays can be generated without error.

## 2022-04-12

@jhchabran

Valery reached me out in DM about [this build](https://buildkite.com/sourcegraph/sourcegraph/builds/141714#aa7664f3-69d2-4c8e-a6e5-fe8c8d5dad6d) and [its PR](https://github.com/sourcegraph/sourcegraph/pull/33701/files) that was behaving strangely. I mentioned him that it's better to reach out over our Slack channel instead btw. We're getting a cache hit as expected, but somehow yarn still wants to download things, ending up taking about 2/3 minutes whereas it should have been less than ten seconds like it does with the normal caching. We tried to reproduce it locally, but stopped when Valery saw that rebasing the `main` branch fixed the problem. We still took a look at what was happening in the PRs but did not find any relevant reason. This means that we have an edge case where it's possible to hit the cache but still have a slow build. The FP Team is aware of it, and they'll monitor it. I also showed Valery how to search in Grafana to find out occurences of Percy (the pupetteer tests) failing. $DURATION=30m

I inquired about the flakinness of those tests and what they plan to do about it. Valery told me that in Q3 they plan to swap Percy to something else. Due to the immediate impact of the flakiness, I suggested him that we should look into how often those tests are actually bringing forward regressions, which, if not that frequent, means we could turn those tests into soft failures, at least on the `main` branch for the time being, possibly adding a Slack notifications so their team can manually review is something is broken or not. That would bring down the flakes ratio down significantly. They're going to talk about it during they team sync today. $DURATION=10m

## 2022-04-11

@jhchabran

[No healthy agents running](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1649658351891759). Agents are failing to start because of missing volume: `buildkite-git...`. It seems it cannot find the disk attached to the volume. I cannot launch again the pipepline to start fresh because no agents are available. I ended up creating an incident, because of the total disruption of the CI, see: [INC-94](https://sourcegraph.slack.com/archives/C03AHJL049M), which mentions with more details how I fixed it. $DURATION=60m

[Blocked main builds because of mismatch in generated files](https://sourcegraph.slack.com/archives/C02FLQDD3TQ/p1649669568010659). Joe thought it was a flake due to the log uploading failures and missed that there was an error in the generated files. He's opening a PR. $DURATION=5m

## 2022-04-06

@bobheadxi

[Thread discussing slow job setup times](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1649240294407269). We need to mitigate the impact of slow cloning on stateless agents, since it's a huge contributor to slowness. [#30237](https://github.com/sourcegraph/sourcegraph/issues/30237)

[Thread about how draft => ready for review triggers a rebuild](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1649254250381289). Side effect of frontend platform request for ready-for-review-only steps.

Agent on `default` queue [stuck for 14 minutes](https://sourcegraph.slack.com/archives/C02KX975BDG/p1649253119257889?thread_ts=1649252899.879369&cid=C02KX975BDG). Fix: add a configurable secondary queue to job dispatcher, which is used to look for scheduled jobs, and set `queue=default` [infrastructure#3208](https://github.com/sourcegraph/infrastructure/pull/3208) $DURATION=30m

[Discussion about `queue` usage](https://sourcegraph.slack.com/archives/C07KZF47K/p1649272455134259?thread_ts=1648590958.956749&cid=C07KZF47K). Follow-ups: [create pipeline creation guide](https://github.com/sourcegraph/handbook/issues/2993), [revert `job` and `stateless` queues to `standard`](https://github.com/sourcegraph/sourcegraph/issues/33238#issuecomment-1090779452)

## 2022-04-05

@bobheadxi

CI incident raised due to:

- Checkov checks breaking (dependency not found)
- Pupetteer finalize flaking
- Newly introduced Pupetter check flaking (chrome extension)
- npm install flake (503)

@bobheadxi mostly reading logs and pinging teams and asking he flakey steps be disabled. Nothing we can do about the infra flake for now, though [#26257](https://github.com/sourcegraph/sourcegraph/issues/26257) could mitigate. $DURATION=30m

We hit GitHub git-lfs bandwidth limits today. Turns out this was due to `git clone` implicitly downloading LFS assets, instead of only fetching LFS assets on `git-lfs fetch`, causing jobs that don't need LFS assets to download LFS assets - we need to explicitly set `GIT_LFS_SKIP_SMUDGE=1` to disable this behaviour. [#3206](https://github.com/sourcegraph/infrastructure/pull/3206)

Had to disable browser extension tests anyway because LFS ended up causing issues again later.

## 2022-04-04

@bobheadxi

Spotted two flakes. Didn't do anything except note their occurrence.

[build 140335](https://buildkite.com/sourcegraph/sourcegraph/builds/140335#803f8727-c5c0-484d-80fe-96f5b1f3c45f/181-2772), isolated occurrence according to [grafana query](https://sourcegraph.grafana.net/goto/7494m3snz?orgId=1).

```none
START| IndexStatus
     |     dbtest.go:111: testdb: postgres://postgres:sourcegraph@127.0.0.1:5432/sourcegraph-test-3601185618347116340?sslmode=disable&timezone=UTC
     |     store_test.go:524: expected index status
     |     dbtest.go:121: DATABASE sourcegraph-test-3601185618347116340 left intact for inspection
FAIL | IndexStatus (2.04s)
```

[build 140327](https://buildkite.com/sourcegraph/sourcegraph/builds/140327#70b8c0bd-a53c-48e5-a90c-0a3ec2bf704e/216-294), nothing we can do here.

```none
error An unexpected error occurred: "https://registry.npmjs.org/@xstate/fsm/-/fsm-1.4.0.tgz: Request failed \"522 undefined\"".
```

Check output and splitting steps so that the checks they run are more granular (most notably prettier, yarn). https://github.com/sourcegraph/sourcegraph/issues/33363

[Thread about submodule cloning](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1649074147208429). [Buildkite discussion about disabling submodule fetch by default](https://github.com/buildkite/agent/issues/1053#issuecomment-989784531), potential workaround, though not implementing in favour of removing submodules entirely: [#33384](https://github.com/sourcegraph/sourcegraph/issues/33384)

Not part of devx-support, but @bobheadxi investigated the `prom-wrapper` alerting configuration issue raised in [INC-93](https://app.incident.io/incidents/93) ([writeup](https://github.com/sourcegraph/sourcegraph/issues/33394)) with the help of @michaellzc, the root cause of which turned out to be some [DevX work to refactor the `conf` package's `init` behaviour](https://github.com/sourcegraph/sourcegraph/issues/29222). The fix: https://github.com/sourcegraph/sourcegraph/pull/33398 , Delivery is doing the remainder of the investigation and follow-up. $DURATION=90m

## 2022-04-01

@bobheadxi

Frontend percy finalize failing consistently with no output:

```none
npm WARN exec The following package was not found and will be installed: @percy/cli
Error: exit status 1
```

- https://buildkite.com/sourcegraph/sourcegraph/builds/140259#7cd6556a-83e3-4c52-bd33-4934fe072385/159-163
- https://buildkite.com/sourcegraph/sourcegraph/builds/140241#025449b8-bc41-43dd-b9ca-1c26d89ce41e/165-169
- Also seen on PR branches, e.g. https://buildkite.com/sourcegraph/sourcegraph/builds/140249#7cfc48e9-637b-468e-b0e8-7483075a2ce5/161-165

Made a request for help to frontend platform: https://sourcegraph.slack.com/archives/C01LTKUHRL3/p1648856877518069 $DURATION=30m

Opened a PR to try and reduce [confusion around unrelated failures](https://sourcegraph.slack.com/archives/C02FLQDD3TQ/p1648851816913459): https://github.com/sourcegraph/sourcegraph/pull/33326

## 2022-03-31

@jhchabran

Spotted this [exception](https://github.com/sourcegraph/sec-pr-audit-trail/issues/144) that should not exist as the base branch is not `main`. Led to create [this bug report](https://github.com/sourcegraph/sourcegraph/issues/33275). $DURATION=5m

## 2022-03-30

@davejrt, @bobheadxi, @jhchabran

[This](https://github.com/sourcegraph/infrastructure/pull/3191) PR caused a small outage when we didn't merge in the correct order, and part of the core job was overwritten by the CI on the infrastructure repo. We had to do some juggling between stateless agent rollout, and merging this PR to unblock a large queue of other jobs that accumulated whilst stateless agents wouldn't start.

@bobheadxi set up an alert on `seconds waited for expected dispach` (`secondsWaitedForExpectedDispatch`), which will notify us in #dev-experience-internal if the jobs dispatched by the dispatcher do not roll out sufficiently within the expected time frame. Visible in the [dispatcher dashboard](https://console.cloud.google.com/monitoring/dashboards/builder/a87f3cbb-4d73-476d-8736-f3bc1ca9f234?folder=true&organizationId=true&project=sourcegraph-ci&dashboardBuilderState=%257B%2522editModeEnabled%2522:false%257D&timeDomain=1h).

## 2022-03-29

@davejrt, @bobheadxi, @jhchabran

- `depguard` issues on both stateful and stateless agents with more frequency that is now becoming more disruptive. Initial investigation hasn't come up with anything definitive so for the moment, the tests have been disabled [here](https://github.com/sourcegraph/sourcegraph/pull/33182) and an [issue created](https://github.com/sourcegraph/sourcegraph/issues/33183)

## 2022-03-28

@jhchabran

It was reported that `sg` was stuck in an autoupdate loop. This was caused by
the fact that the user had two `sg` installed: one under `~/.sg/sg` and the other one in the `$GOBIN` folder. Because the detection is based on what's in the repository, the result was an infinite loop.

My hunch here is that we should just pick one, and always stick to that install location. I asked Thorsten his thoughts [here](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1648468340459119?thread_ts=1648467054.249209&cid=C01N83PS4TU).

## 2022-03-25

@bobheadxi

- Sporadic `codeinsights` test failures from yesterday. Pinged again, fixed by @coury-clark via DB conn bump. [thread](https://sourcegraph.slack.com/archives/C02FLQDD3TQ/p1648184742528449)
- Unlinted code was merged: https://github.com/sourcegraph/sourcegraph/pull/33094
- Incident: GraphQL changes breaking main. Fix: https://github.com/sourcegraph/sourcegraph/pull/33092, proposed mitigation by @jhchabran: https://github.com/sourcegraph/sourcegraph/pull/33095
- `depguard` issues on some agents: intermittent, can occur on new-ish agents. causes a load of `depguard` false positives that fail the build. potentially co-incides with `golangci-lint 1.45.0` upgrade, though issues persisted both before and after downgrade. [thread](https://sourcegraph.slack.com/archives/C02FLQDD3TQ/p1648227503962249)
  - @davejrt suggests we wait for a repeat occurrence and deep-dive on the agent where it happened.
  - https://github.com/sourcegraph/sourcegraph/pull/33118 rolls out 50% so we can hopefully get a clearer comparison. See below

Also note deployment of the new stateless job dispatcher. Links to monitor:

- [Dispatcher logs](https://cloudlogging.app.goo.gl/pqcDLz9aeFCDtWTU9)
- [Dispatched agents](https://console.cloud.google.com/kubernetes/workload/overview?project=sourcegraph-ci&pageState=(%22savedViews%22:(%22i%22:%22d63788ab9603422da3abba5f06030393%22,%22c%22:%5B%5D,%22n%22:%5B%22buildkite%22%5D),%22workload_list_table%22:(%22f%22:%22%255B%257B_22k_22_3A_22Is%2520system%2520object_22_2C_22t_22_3A11_2C_22v_22_3A_22_5C_22False_~*false_5C_22_22_2C_22i_22_3A_22is_system_22%257D_2C%257B_22k_22_3A_22Name_22_2C_22t_22_3A10_2C_22v_22_3A_22_5C_22buildkite-agent-stateless-_5C_22_22_2C_22i_22_3A_22metadata%252Fname_22%257D%255D%22)))
- [Dispatcher metrics](https://console.cloud.google.com/monitoring/dashboards/builder/a87f3cbb-4d73-476d-8736-f3bc1ca9f234?project=sourcegraph-ci)

To roll back, revert https://github.com/sourcegraph/sourcegraph/pull/33107 or change the queue to target `stateless` instead of `stateless2`. Configuration changes are in https://github.com/sourcegraph/infrastructure/pull/3182

Most things can be traced using the dispatched ID, which is included in the job name, agent metadata, and log fields - e.g. to trace down why the dispatcher decided to make a dispatch.

## 2022-03-22

@bobheadxi

- Node installation failures in deploy-sourcegraph-cloud pipeline [thread](https://sourcegraph.slack.com/archives/C02KX975BDG/p1647971014157899?thread_ts=1647957942.367989&cid=C02KX975BDG)
  - Might be caused by really old Node version in the pipeline, or just plain bad state. Doesn't seem to be happening on the sourcegraph pipeline
  - @daxmc99 deletes a bunch of stuff propagated from upstream depoy-sourcegraph, as well as the prettier step entirely: https://github.com/sourcegraph/deploy-sourcegraph-cloud/pull/15935

## 2022-03-21

@bobheadxi, @jhchabran

We have been running into sporadic, severe issues with stateless agent availability since we [rolled out stateless builds to 75% of `sourcegraph/sourcegraph` builds](https://github.com/sourcegraph/sourcegraph/pull/32751). See [thread](https://sourcegraph.slack.com/archives/C02MWRMAFR8/p1647838869486089). Caused some severe downtime in the morning, with [incident 92](https://app.incident.io/incidents/92)

- Increased `backoffLimit` to max: https://github.com/sourcegraph/infrastructure/pull/3176
- Discussed autoscaling implementation, landed on some misunderstandings: https://github.com/sourcegraph/infrastructure/pull/3177
- Scaled down agent rollout top 10%: https://github.com/sourcegraph/sourcegraph/pull/32840

Follow-up: potential revamp of autoscaler. https://github.com/sourcegraph/sourcegraph/issues/32843
