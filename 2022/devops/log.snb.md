# 2022 DevOps log

Log for DevOps tasks handled by DevX teammates, old and new.
To add an entry, just add an H2 header with ISO 8601 format.
The first line should be a list of everyone involved in the entry.
For ease of use and handing over issues, **this log should be in reverse chronological order**, with the most recent entry at the top.

## 2022-09-08 Deploying a scale testing cluster on GCP, part 2

@sanderginn @jhchabran

After @mohammadualam unblocked me on the permissions issue, we could resume creating the infrastructure. 

Some gotchas:
- The `node_locations` variable of the `terraform-google-sourcegraph-gke` module should just be set to `null` if there's no need for a multi-zonal cluster. If you use the `google_container_cluster` resource directly you would just omit this var as it is optional, but the module requires you to pass _something_ for it.
- Cloud SQL can only use predefined VM types with shared cores (e.g. `micro`-type VMs) _or_ custom VM types with `db-custom-<cores>-<mem in mb>` syntax. Wtf.

After the cluster spun up, we moved on to installing SG. We used the Helm install guide, starting with the GCP/GKE section. The configuration is checked into [deploy-sourcegraph-scaletesting](https://github.com/sourcegraph/deploy-sourcegraph-scaletesting).


To mirror our existing setups, we installed Nginx as an ingress controlller. We simply copied the helm definition from `deploy-sourcegraph-dogfood-k8s`. To get it to install we needed to create the namespace `ingress-nginx`. This lead to a new problem: installing the Nginx helm chart tried to delete a role on the cluster, which requires permissions that we do not have by default. After being granted the role `Kubernetes Engine admin` on this project only, we could install the helm chart for the Nginx Ingress controller.  

We created the namespace `scaletesting` with a plain `kubectl create ns scaletesting`.

The next step was to ensure the secrets are all present on the cluster. We copied `gsm-secrets.tf` from dogfood in the infrastructure repository. When applied, these will properly import the secrets from GSM into the cluster, but they still need to be manually created in GSM. 
The following secrets were made:

- `codeintel-bucket-sa-secret`: went to https://console.cloud.google.com/apis/credentials/serviceaccountkey and created a key for the SA (json format), then added it to GSM with the expected key.
- `kms-service-account-key`: same as above, but other projects don't capture this SA in Terraform (sigh), so first added an extra resource to create the SA to `main.tf`.
- `minio-secrets`: added default secret/key for minio installs.
- `sgdev-tls`: copied secret values from dogfood's GSM (they were identical to the cert data in buildkite so presume this is OK).


To apply the secrets, the `kubernetes` provider block needed to be updated to a newer version:

```terraform
provider "kubernetes" {
  host = "https://${module.gke.gke_master_ip}"

  cluster_ca_certificate = base64decode(
    module.gke.gke_cluster_ca_certificate,
  )
  token = data.google_client_config.current.access_token
}
```

After applying all the secrets, we copied the `values.yaml` for Sourcegraph's helm chart from `deploy-sourcegraph-dogfood-k8s` and started commenting out everything that was not applicable (yet). With a bit of fiddling here and there we got it to install (`helm upgrade --install --values ./helm/sourcegraph/override.yaml --version 3.43.1 sourcegraph sourcegraph/sourcegraph`). Only `prometheus` is stuck in `ContainerCreating` because its config does not exist and is trying to be mounted.

What is left to do (minus the things that I am undoubtedly forgetting about):
- Cloud SQL users + permissions need to be set.
- Databases need to be created and hooked up with Cloud SQL proxy (I think?) to their respective applications. 
- Executors are not yet enabled. If that needs to happen, there's a [repo](https://sourcegraph.sourcegraph.com/github.com/sourcegraph/terraform-google-executors) for it. The accompanying secrets need to be added in GSM, enabled in `gsm-secrets.tf`, and the volumes/volumeMounts need to be commented in (lol) again.
- A DNS record for `scaletesting.sgdev.org` still needs to be added (I am fairly confident that `frontend` and the ingress controller are configured properly now, but it might be a good idea to look through the helm values of nginx to see if there are any references to dogfood or `k8s.sgdev.org` still. The certs are mounted in frontend at least).
- Code hosts set up
- Site config config'd
- Users added

This install is of `3.43.1` so does not use any of the OpenTelemetry middleware yet, just plain old jaeger goodies. I'm not sure how easy it is to install just the head of the helm repo instead of a release? Probably really easy, but today has not been a day of easy things, so who knows.

Notes:
- As suggested by @jhchabran we should come up with a mechanic that puts the cluster to sleep. 


## 2022-09-07 Deploying a scale testing cluster on GCP, not unblocked yet

@sanderginn @jhchabran @davejrt @burmudar

We started with copying a lot of the configuration from [`infrastructure/cloud`](https://sourcegraph.sourcegraph.com/github.com/sourcegraph/infrastructure/-/tree/cloud), removing services that are not relevant to the use case of scale testing, and adjusting resources to more cost-appropriate levels.  
A change compared to the source directory is making use of the `terraform-google-sourcegraph-vpc` and `terraform-google-sourcegraph-gke` [modules](https://sourcegraph.sourcegraph.com/github.com/sourcegraph/infrastructure/-/tree/modules).

Running `terraform apply`, we were reminded of the fact that GCP projects are actually maintained in the [`gcp/projects`](https://sourcegraph.sourcegraph.com/github.com/sourcegraph/infrastructure/-/tree/gcp/projects) directory. After adding the new `sourcegraph-scaletesting` project in `terraform.tfvars`, we were not capable of applying the change due to a lack of permissions. Ultimately, it was applied by @filiphaftek the next day, as he has the correct permissions due to the daily operations of the Cloud team.  

Applying the [PR](https://github.com/sourcegraph/infrastructure/pull/3880) with the new cluster definitions still fails, again due to lacking permissions:
```sh
╷
│ Error: googleapi: Error 403: Permission denied to update connection for service 'servicenetworking.googleapis.com'.
│ Help Token: AfeSHlKZcPWgR7VbqlocgmJ3zd_GGBzDffQtWodfGY16UsuRCgZt70OaBt2qy5di44hm4dWRrdT-8mVWuc3SnZVIapePud9scNGAG6cd0xX9A_0-
│ Details:
│ [
│   {
│     "@type": "type.googleapis.com/google.rpc.PreconditionFailure",
│     "violations": [
│       {
│         "subject": "?error_code=110002\u0026service=servicenetworking.googleapis.com\u0026permission=servicenetworking.services.addPeering\u0026resource=446689196375",
│         "type": "googleapis.com"
│       }
│     ]
│   },
│   {
│     "@type": "type.googleapis.com/google.rpc.ErrorInfo",
│     "domain": "servicenetworking.googleapis.com",
│     "metadata": {
│       "permission": "servicenetworking.services.addPeering",
│       "resource": "446689196375",
│       "service": "servicenetworking.googleapis.com"
│     },
│     "reason": "AUTH_PERMISSION_DENIED"
│   }
│ ]
│ , forbidden
│ 
│   with google_service_networking_connection.private_vpc_connection,
│   on sql.tf line 22, in resource "google_service_networking_connection" "private_vpc_connection":
│   22: resource "google_service_networking_connection" "private_vpc_connection" {
```

After some attempts to resolve this with @diegocomas it was not solved yet, but I did find the root cause:

* To correctly [configure a private vpc connection](https://cloud.google.com/vpc/docs/configure-private-services-access#permissions) the role `roles/compute.networkAdmin` is required.
* The `gcp-devex@sourcegraph.com` group has the role `Sourcegraph Org Editor` on the project folder `Sourcegraph Cloud`:
![DevX GCP role](https://raw.githubusercontent.com/sourcegraph/devx-scratch/6e08c098a4b58ad4a2eac6f54e9c174c10220735/2022/devops/gcprole.png)
* The definition of this custom role does not include `roles/compute.networkAdmin` right now and needs to be expanded:

https://sourcegraph.sourcegraph.com/github.com/sourcegraph/infrastructure/-/blob/gcp/org/sg-editor.tf?L2

A side note here is that it is not entirely clear if any new projects we create are automatically added to the project folder `Sourcegraph Cloud`, which is necessary for us to inherit our permissions on the new project. Based on the first Terraform runs, I do think this happens at project creation already.

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
