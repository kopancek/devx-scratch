# 2022 DevOps log

Log for DevOps tasks handled by DevX teammates, old and new.
To add an entry, just add an H2 header with ISO 8601 format.
The first line should be a list of everyone involved in the entry.
For ease of use and handing over issues, **this log should be in reverse chronological order**, with the most recent entry at the top.

## 2022-08-31 Deploying otel-collector to sourcegraph.com

@bobheadxi @sanderginn @marekweb @davejrt @burmudar

Started with following the guidance provided [here](https://handbook.sourcegraph.com/departments/engineering/dev/process/deployments/#merging-upstream-deploy-sourcegraph-into-deploy-sourcegraph-forks), but turned out the `master` branch tracking upstream was deleted so we had to recreate it:

```sh
git fetch upstream
git checkout --track upstream/master
```

The process from then on was a complete disaster, the dotcom has diverged a lot from upstream and it was very hard to reconcile, so we just gave up and applied the relevant diffs in https://github.com/sourcegraph/deploy-sourcegraph/pull/4163 by hand.

For parity with existing Grafana Cloud tracing in dotcom, we followed [this guide](https://grafana.com/blog/2021/04/13/how-to-send-traces-to-grafana-clouds-tempo-service-with-opentelemetry-collector/) to update the OpenTelemetry collector exporter configuration:

```sh
    exporters:
      # Send to Grafana Cloud Tempo
      otlp:
        endpoint: tempo-us-central1.grafana.net:443
        headers:
          authorization: Basic $GRAFANA_OTLP_AUTH
```

Where we get the credentials from [Grafana Cloud -> Configuration -> Data sources -> Tempo](https://sourcegraph.grafana.net/datasources/edit/grafanacloud-traces):

```sh
GRAFANA_OTLP_AUTH=$(echo -n "<your user id>:<your Tempo api key>" | base64)
```

We added this to the `grafana-agent` secrets [via Terraform](https://github.com/sourcegraph/infrastructure/commit/0099f712cf81e9197a9b62d7666320f2f32da16f), and configured the OpenTelemetry collector to read from it:

```yaml
    spec:
      containers:
        - name: otel-collector
          # ...
          env:
            - name: GRAFANA_OTLP_AUTH
              valueFrom:
                secretKeyRef:
                  name: grafana-agent-secrets
                  key: GRAFANA_OTLP_AUTH
```

A critical mistake comes when applying the `configure/otel-collector` manifests with the ConfigMap added in https://github.com/sourcegraph/deploy-sourcegraph/pull/4163 in dotcom, https://github.com/sourcegraph/deploy-sourcegraph-cloud/pull/17254/files#diff-03b00da2613f69a259341167e857a9a508b0d08258f3b69d45ce7c9aa078f9c2, where I copy paste an existing line:

```diff
# Deploy kube-dns
apply configure/kube-dns "$(maybe_exclude_rbac deploy=sourcegraph)"

+ # Apply OpenTelemetry collector configuration
+ apply configure/otel-collector "$(maybe_exclude_rbac deploy=sourcegraph)"
```

This was a grave mistake, and lead to [INC-141](https://app.incident.io/incidents/141) - see the soon-to-be-updated [INC-141 postmortem](https://docs.google.com/document/d/1wF2hhF5LY47oEsW6JkTqhQ25meazv3yqrsK-A7w9nw4/edit#) for more details.

## 2022-08-03 add devx permissions to upgrade s2

@kalanchan

https://github.com/sourcegraph/infrastructure/pull/3720
https://github.com/sourcegraph/deploy-sourcegraph-managed/pull/814

todo:

- add workload identity service account to be able to upgrade s2

## 2022-07-04 deploy-sourcegraph-cloud renovate scheduling

@bobheadxi

It was noted that Grafana and Prometheus have not been updated in several weeks.
I think the main issue is that the schedule was provided in a crontab format, when [Renovate scheduling docs make no indication it supports crontab](https://docs.renovatebot.com/key-concepts/scheduling/) - the syntax seems documented here: https://breejs.github.io/later/parsers.html#text

I decided to just push directly to the preprod branch with some fixes (pull requests are a nightmare to merge): https://github.com/sourcegraph/deploy-sourcegraph-cloud/pull/17105/commits/24f9d5efb82d81a2a307b2a2b5ff4c9b2a33d38e , tl;dr

```diff
-     schedule: ["* 15 * * 1-5"],
+     schedule: ["after 3pm on monday"],
```

It also seems the schedule has changed - what used to by "Daily" is actually weekly, and so on, so I've updated the names of the Renvoate jobs as well. I then manually bumped Prometheus and Grafana to `158142_2022-07-04_0ce424594726` by hand in pre-prod.

All this will be merged in https://github.com/sourcegraph/deploy-sourcegraph-cloud/pull/17105

## 2022-07-04 GCP Cost Reduction and CD improvements

@kalanchan

Working with Daniel and Manny to identify areas of improvement to dotcom and preprod deployments

- TLDR; commits in preprod branch get squashed and causes conflicts when more image tags get changed and need to be merged into release(main) again.
- Potential fix wouild be to run Github Actions that creates a brand new branch on run time instead of having renovate make updates to preprod. Could incorporate `sg` usage here instead of writing custom script. Dax has some progress [here](https://github.com/sourcegraph/deploy-sourcegraph-cloud/pull/16390), but it's not perfect due to formatting/linting issues.

Regarding GCP cost reductions, we'll tackle the low hanging fruit first identified by RafG in his [doc](https://docs.google.com/document/d/1FQScUkS6fyBfW__dG0WiHmXH6-Fl_4zWwMTY9IWASSI/edit#heading=h.na988urmj90p).  
[Older Version](https://docs.google.com/document/d/1qEnD-1RQ0tD_C-kKiLngKnWA1-kUStOu4xU0YGlKQrM/edit#heading=h.m2y4u5mmwaiw)

- get understanding of usage vs provisioned resources
- identify peak periods and consumption
- involve targetted service's stakeholders to see if we can reduce infrastructure provisioning
- build foundations for FinOps. [FinOps is to Dollars What DevOps is to Code](https://devops.com/how-finops-can-optimize-cloud-costs-and-drive-innovation/)
- [eventual "observability" into cloud cost](https://cloud.google.com/blog/topics/developers-practitioners/optimizing-your-google-cloud-spend-bigquery-and-looker)

## 2022-07-04 Dotcom crashed due to invalid site config

@sanderginn

Opened a [PR](https://github.com/sourcegraph/deploy-sourcegraph-cloud/pull/17063) addressing invalid JSON and YAML that gets committed and deployed. In this case, it led to brief interruptions of sourcegraph.com.
