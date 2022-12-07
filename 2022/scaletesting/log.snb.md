# 2022 scale testing log

DevX teammates and teammates hacking on Sourcegraph's scaletesting set of tools.

To add an entry, just add an H2 header starting with the ISO 8601 format, a topic.
**This log should be in reverse chronological order.**

## 2022-12-07

@milan
Found out, that permission syncing is still slow. Added the following to the site-config:
```
"permissions.syncUsersMaxConcurrency": 8,
```

Played with this value a bit and restarted the `repo-updater` pod a few times (by deleting it).

Waited a few hours, but noticed that permissions are not updating at all. After some investigation, I found out on [Outbound requests page](https://scaletesting.sgdev.org/site-admin/outbound-requests) that there is a user that requests 190 pages of repositories, so it seemed like the user had 200k repositories attached on ghe instance.

Found out the user is `dev-experience-team@sourcegraph.com`, promoted the user to site-admin and purged it's records from the `user_permissions` table.
Restarted the pod again and now it seems like perms syncing is being properly scheduled again.


## 2022-12-06

@milan
Removed enforcement of permissions from the [Perforce code host](https://scaletesting.sgdev.org/site-admin/external-services/RXh0ZXJuYWxTZXJ2aWNlOjI1)
Purged tables from the Postgres DB:
```
repo_permissions
user_permissions
repo_pending_permissions
user_pending_permissions
sub_repo_permissions
```
Added enforcement of permissions to the [GHE Scaletesting code host](https://scaletesting.sgdev.org/site-admin/external-services/RXh0ZXJuYWxTZXJ2aWNlOjI2)

Waited for an hour and looked at the metrics. 

Stopped enformcement of permissions and then added the following to the site-config:
```
"permissions.syncOldestRepos": 1,
"permissions.syncOldestUsers": 150,
```

Purged the DB tables again and restarted the enforcement of permissions

## 2022-11-04

@davejrt the self-signed TLS certificiate for [ghe-scalingesting.sgdev.org](https://ghe-scaletesting.sgdev.org) was blocking Indra and the repo management team. Previous attempts to enable `Let's Encyrpt` signed certificates on the host were returning two errors, below: 

```
Running `sudo -u admin /usr/local/bin/ghe-ssl-acme -p -i` and writing log to /tmp/acme-issue.log.20221104-3297-pxs2n8, pid=5855
---> Running sudo -u acme-client acme.sh --allow-sudo --syslog 6 --config-home /tmp/tmp.acme-workdir.nCSLvqUty3  --issue --stateless -d ghe-scaletesting.sgdev.org
[Fri 04 Nov 2022 12:04:17 PM UTC] Domains not changed.
[Fri 04 Nov 2022 12:04:17 PM UTC] Skip, Next renewal time is: Mon 02 Jan 2023 06:25:19 PM UTC
[Fri 04 Nov 2022 12:04:17 PM UTC] Add '--force' to force to renew.
---> End (failed: 2)
ghe-ssl-acme error: The certificate at /tmp/tmp.acme-workdir.nCSLvqUty3/ghe-scaletesting.sgdev.org/fullchain.cer is not present; did ACME fail?
```

and the following error in a banner when trying to save changes in the management console
```
"Github ssl cert The certificate is not signed by a trusted certificate authority (CA) or the certificate chain is missing intermediate CA signing certificates."
```

I took an educated guess and logged into the applicance, and ran the commands I found in [this documentation](https://docs.github.com/en/enterprise-server@3.6/admin/configuration/configuring-your-enterprise/command-line-utilities#ghe-ssl-acme)

Despite previous attempts to sign install the let's encrypt signed cert via the UI, I was still seeing the self-signed certificate when running `ghe-ssl-acme -s`. I was able to remove this cert by running `ghe-ssl-acme -x`, and generate a new Let's Encrypt certificate by running  `ghe-ssl-acme -e -v` which was successful. I verified this via the cli with the snippet below as well as in the browser. 

```
admin@ghe-scaletesting-sgdev-org:~$ ghe-ssl-acme -s
SSL enabled:           true
Active certificate CA: Let's Encrypt
ACME enabled:          true
ACME provider:         letsencrypt
ACME ToS accepted:     true
ACME Contact E-mail:
ACME Account Key:      (key is set; retrieve with `ghe-config "secrets.acme.account-key"`)
ACME CA conf string:   (CA conf is set; retrieve with `ghe-config "github-ssl.acme.ca-conf"`)
```

## 2022-10-26

@burmudar so I got access to the EKS cluster on AWS to transfer some Bitbucket repos out. Accessing the cluster rightfully so, isn't exactly straight forward. For any changes that require access to the kube API you first need to add **your** ip to the allowed ip's in the advanced networking panel. Now that you've got kube API access all is good right ? WRONG.

You can change the cluster. Add services, expose some endpoints, add pods, remove pods but the IP you added earlier is **only** for kube API access. Ok, so knowing this, let's ask Google "How do I expose a Pod on EKS". All I'm really looking for is just some guidance on setting up a quick and dirty thing. Something slightly higher than `kubectl port-forward`. Instead, all the resources are about adding a Cloud Load Balancer, adding ingress rules, ingress classes and I was really not to keen on doing any of that since it is aimed at really running a service. I had some vague recollection about NodePorts, and eventually I came across the command `kubectl expose` which allows you to quickly create a service and expose some pod.

With my new found knowledge of `kubectl expose` I started exposing services targeting the bitbucket pod and eventually settled on the following command `kubectl expose pod bitbucket-0 --port=7990 --target-port=7990 --name bitbucket-svc --type NodePort --external-ip=<my external ip>`. This created a service which exposed a NodePort on the node of the `bitbucket` pod was running on. With the service created I fired up my trusty curl to `<external ip>:7990` ... and get a timeout. Looking closely at the service output, I saw that the ports that are mapped are a bit different to how I specified them on the cli `7990:30186/TCP`. Reading as to why this happens and it seems EKS has some restriction to only expose NodePorts in the 30000 range. Moving on ... I fired up curl again with the new port ... and we got a nice fat timeout again. I'm pretty sure the kubernetes config is correct (to my laymans brain).

So I went digging, and eventually I struck gold, by looking at the security allow list of the VPC that is used in AWS for the cluster. I saw Idra had a rule to allow postgres, so I added a rule to allow inbound traffic on port `30186`. Once the rule got assigned and I fired off a curl ... SUCCESS! I got the bitbucket dashboard response.

## 2022-10-21

@jhchabran, about generating 200k blank repos: the 200k blank (one commit with a README.md) repos creating is progressing at 26 repos / minute.  All repos on GH have been created, it's just the push part that takes time.

This is slow. It should take another 4 days to complete the remaining 170k blank repos.

With regard to the timeline to provide test-data, it's better than nothing. But I suspect this is only going to become a bigger problem in the future, especially when we're trying to reproduce a customer issue (which is the case here).

Actually, I should ask a GitHub solution architect about this.

## 2022-10-17

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

## 2022-10-11

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


