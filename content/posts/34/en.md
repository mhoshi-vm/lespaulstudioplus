---
title: "Trying Tanzu Platform Self Managed — Installation"
date: 2025-01-05T21:34:12+09:00
tags: ["Tanzu Platform"]
thumbnail: "aeba8b9e.png"
featured: true
---

Let's try the on-prem edition of the latest product, Tanzu Platform. This time we won't follow the official steps — we'll use a custom installer I built.
<!--more-->

# Series

- **Here>** Installation
- [For admins: Project setup](../35)
- [For users: Deploying to a Space](../36)
- [For admins: Slimming down the deploy target](../37)
- [For admins: Enabling HTTPS](../38)
- [For admins: Automatic DNS registration](../39)

Bonus: [The "Things You Don't Need to Know about Tanzu Platform" series](/categories/tanzu-platform-for-maniacs/)


# How to Install(Official)

For the regular installation steps, see:

https://techdocs.broadcom.com/us/en/vmware-tanzu/platform/tanzu-platform/10-0/tnz-platform/tp-sm-install-install-tp-sm.html

Following the regular installation steps you run into these problems:
- Enormous resources are required
- Many installation steps
- Uninstalling isn't casual (the uninstaller does extra work purging PVs, and full deletion takes nearly an hour)

# How to install(Unofficial)

So I built a simple installer for studying TP. See the README for the steps. Please understand this was built for verification while still getting familiar with TP.

https://github.com/mhoshi-vm/tap-carvel/tree/pkgr/manifests/tpk8s-opinionated.tanzu.japan.com

As written in the README, the prerequisites are:

- A TKGs/TKGm environment
- At least 1 controller; confirmed working at 2vCPU/4GB
- At least 2 compute machines of 10vCPU/32GB/200GB. With 1 machine you get a "too many Pods to fit" error. That's a lot of resources, but it is still less than 2/3 of the official requirement.
- Not tested outside TKGs/TKGm, but as long as Cluster Essentials is installed it should work under the same conditions.

Installation wait time is a few dozen minutes.
If it goes well, it installs in force, like this:

```
mh013301@PJQ72XCV5C ~ % kubectl get po -n tanzusm
NAME                                                    READY   STATUS      RESTARTS        AGE
account-manager-server-6947ccf46c-m54xt                 1/1     Running     5 (7m12s ago)   36m
agent-gateway-server-79c785b8c4-ck6nt                   1/1     Running     0               36m
alertmanager-tmc-local-monitoring-tmc-local-0           2/2     Running     0               35m
api-gateway-server-685dfc544b-flgcq                     1/1     Running     0               36m
aria-sync-service-server-575df8bd48-ppx6t               1/1     Running     2 (37m ago)     37m
authentication-server-84c56bf94-vwxhj                   1/1     Running     0               37m
ccc-derived-data-service-65d5945ff5-pl8tk               1/1     Running     0               53m
ccc-insights-service-55457d995c-9l2v7                   1/1     Running     0               53m
ccc-insights-service-topic-hook-job-lvv4m               0/1     Completed   0               53m
ccc-rules-service-9d8895c65-pc7pk                       1/1     Running     0               53m
ccc-rules-service-topic-hook-job-2dc52                  0/1     Completed   0               53m
clickhouse-shard0-0                                     1/1     Running     0               68m
cloud-accounts-service-547d9f957b-dtsvh                 1/1     Running     0               47m
cluster-agent-service-server-7c769d4c57-cnhs5           1/1     Running     0               36m
cluster-config-server-7c66bf9dcc-fc5r6                  1/1     Running     0               36m
cluster-object-service-server-5bd9988b9-5bx2d           1/1     Running     0               36m
cluster-reaper-server-7f75d54c4-bj5cg                   1/1     Running     0               37m
cluster-secret-server-b54f87f6f-8m9gg                   1/1     Running     0               36m
cluster-service-server-59d7bf475-z8qxn                  1/1     Running     0               36m
cluster-sync-egest-c89796758-fjllx                      1/1     Running     0               37m
cluster-sync-ingest-695c8cfbf7-89s2n                    1/1     Running     0               37m
contour-contour-6cb6ffb67d-xhkrd                        1/1     Running     0               68m
contour-contour-certgen-h29h8                           0/1     Completed   0               68m
contour-envoy-f4769b5f8-b27p2                           2/2     Running     0               68m
daedalus-674b75b88b-j66fj                               1/1     Running     0               62m
daedalus-trivy-748bcdfdd4-47tqt                         1/1     Running     0               62m
ensemble-app-manager-848d5dcbd7-74pb5                   1/1     Running     0               46m
ensemble-app-metadata-ingestion-68847dc4dd-sqx4x        1/1     Running     0               46m
ensemble-app-metadata-ingestion-lemans-hook-job-j7qkq   0/1     Completed   0               46m
ensemble-application-metadata-59b6cdd5d4-n7jvb          1/1     Running     0               46m
ensemble-build-service-596f9b588b-vzw5x                 1/1     Running     0               46m
ensemble-derived-data-678c5f655b-9j85v                  1/1     Running     1 (40m ago)     46m
ensemble-endpoint-manager-5864c79875-4gb9r              1/1     Running     0               46m
ensemble-inventory-service-67476947c9-hm6qs             1/1     Running     0               46m
ensemble-networking-b5989fb58-lvtrr                     1/1     Running     0               46m
ensemble-notifications-service-c978fc9ff-t7ncb          1/1     Running     0               46m
ensemble-observability-87dd8f448-z9gzj                  1/1     Running     0               46m
ensemble-observability-store-66cc56cdc6-s6q7f           1/1     Running     0               46m
ensemble-observability-store-ingest-697b96d64b-s2d5l    1/1     Running     0               46m
ensemble-observability-store-lemans-hook-job-l6r9n      0/1     Completed   0               46m
ensemble-policy-providers-service-565c8d5d75-lvfkg      1/1     Running     0               46m
ensemble-provider-service-56fb597d6b-4b9c9              1/1     Running     0               48m
ensemble-provider-service-lemans-hook-job-2c7ls         0/1     Completed   0               48m
ensemble-tac-694cdb8957-t5p99                           1/1     Running     0               46m
ensemble-ucp-7b549848bd-d5vdb                           1/1     Running     0               46m
ensemble-ui-6646784959-mk9fc                            1/1     Running     0               46m
ensemble-user-service-85cd65d79d-mlqlf                  1/1     Running     0               46m
events-service-consumer-bcddd775d-vn8vk                 1/1     Running     0               37m
events-service-server-7f88cb6cc9-xmx4b                  1/1     Running     0               37m
fanout-service-server-74cb485597-5dtt2                  1/1     Running     0               36m
feature-flag-service-server-9bdf45cf6-tbbn8             1/1     Running     0               37m
findings-67f988d8c7-4sp6c                               1/1     Running     0               59m
graphql-eds-service-6c8b99779f-26vx7                    1/1     Running     0               46m
graphql-rest-provider-service-5bb79dfbc8-qpzlw          1/1     Running     0               46m
graphql-rest-provider-service-lemans-hook-job-q66wh     0/1     Completed   0               46m
graphql-stitching-service-79d48977fb-qvfnc              1/1     Running     0               46m
helm-deployment-server-5b46d857f6-nzpxs                 1/1     Running     0               36m
index-history-wml45                                     0/1     Completed   0               59m
ingestion-58ffb77db4-k7xf6                              1/1     Running     0               53m
intent-server-769b65955c-jdr8b                          1/1     Running     0               37m
inventory-56c98d5bb6-vltgn                              1/1     Running     0               59m
inventory-cleanser-7cbbdb8fdc-bn2ct                     1/1     Running     0               53m
inventory-consumer-kafka-cloud-f85b5b85d-wtkdk          1/1     Running     0               53m
inventory-service-kafka-hook-job-5sv4n                  0/1     Completed   0               59m
k8s-ingestion-service-lemans-d7f9f4959-br7vt            1/1     Running     0               53m
k8s-ingestion-service-lemans-hook-job-vkbfb             0/1     Completed   0               53m
kafka-topic-controller-7cf5dc96c-2qdxv                  1/1     Running     0               53m
lemans-gateway-hsm-cluster-1-866687cf64-p6qdc           1/1     Running     0               66m
lemans-resources-7d689c64c5-j9fbx                       1/1     Running     0               64m
lemans-resources-post-upgrade-api-job-pxwgb             0/1     Completed   0               64m
onboard-cas-fkbf9                                       0/1     Completed   0               59m
onboard-partitions-x6cbg                                0/1     Completed   0               59m
onboard-scheduler-sbzlk                                 0/1     Completed   0               59m
onboard-system-4bhgn                                    0/1     Completed   0               59m
opensearch-coordinating-0                               1/1     Running     0               66m
opensearch-data-0                                       1/1     Running     0               66m
opensearch-ingest-0                                     1/1     Running     0               66m
opensearch-master-0                                     1/1     Running     0               66m
ops-kafka-0                                             1/1     Running     0               67m
ops-zk-0                                                1/1     Running     0               68m
ops-zk-cluster-status-check-n6xvz                       0/1     Completed   0               67m
package-deployment-server-588859c9c9-zfwr4              1/1     Running     0               36m
partner-gateway-server-688f78765f-hcbq4                 1/1     Running     0               37m
policy-engine-server-67656b4865-j82rl                   1/1     Running     0               36m
policy-insights-server-779fdc8f84-8664p                 1/1     Running     0               36m
policy-sync-service-server-c7d7d77f4-8njxj              1/1     Running     0               37m
policy-view-service-server-5f66bc96d7-ls8wh             1/1     Running     0               37m
postgres-endpoint-controller-6b6b8975b4-5wsbs           1/1     Running     0               54m
postgresql-0                                            2/2     Running     0               67m
prometheus-server-78c5ddfc8d-kgqkb                      1/1     Running     0               66m
prometheus-server-tmc-local-monitoring-tmc-local-0      2/2     Running     0               35m
provisioner-service-server-7fbb6d4678-tdrz8             1/1     Running     0               37m
redis-master-0                                          2/2     Running     0               67m
reloader-reloader-79f5f58749-rztw4                      1/1     Running     0               54m
resource-manager-server-74ddffc95d-dzztz                1/1     Running     0               37m
resource-manager-server-74ddffc95d-xg5kr                1/1     Running     0               37m
rsa-create-vxp4r                                        0/2     Completed   0               72m
seaweedfs-filer-0                                       1/1     Running     0               63m
seaweedfs-master-0                                      1/1     Running     0               63m
seaweedfs-s3-5d68dc65b4-kz8hz                           1/1     Running     0               63m
seaweedfs-volume-0                                      1/1     Running     0               63m
settings-service-server-78664bfd99-6sttl                1/1     Running     0               36m
spring-ingestion-service-6bcb9f45c5-l96pg               1/1     Running     0               46m
spring-ingestion-service-lemans-hook-job-6wtnf          0/1     Completed   0               46m
tas-ingestion-service-lemans-86f8f4456-pfz92            1/1     Running     0               53m
tas-ingestion-service-lemans-hook-job-7gq5w             0/1     Completed   0               53m
telemetry-event-service-consumer-669b86d7fb-pn96j       1/1     Running     0               37m
tpsm-reloader-reloader-7dc95d5687-rcwfb                 1/1     Running     0               68m
uaa-5858d6d8b8-sbjdw                                    1/1     Running     0               62m
ucp-api-64c666cf9d-79tsx                                1/1     Running     0               57m
ucp-core-controllers-cf8974644-scg6w                    1/1     Running     0               55m
ucp-envoy-5cb5954b5d-6d44x                              1/1     Running     1 (54m ago)     55m
ucp-ingestion-service-58f6676fbc-z5jkm                  1/1     Running     0               58m
ucp-ingestion-service-lemans-hook-job-2m4nc             0/1     Completed   0               58m
ucp-kine-765c79d575-hnsbp                               1/1     Running     0               57m
ucp-kine-765c79d575-z7sx2                               1/1     Running     0               57m
ucp-project-syncer-6bb4f7c658-jktzh                     1/1     Running     0               55m
ucp-runtime-controllers-6994d5cd75-7l64m                1/1     Running     0               55m
ucp-token-authn-7979cccfbb-cssd9                        1/1     Running     0               55m
ucp-tokengen-job-wx7kh                                  0/1     Completed   0               7m
ucp-xds-onb-api-5d5d5c-zb72x                            1/1     Running     0               55m
unified-cluster-onboarding-server-79dc5fcfcf-tpk6t      1/1     Running     0               37m
vss-cloud-accounts-service-lemans-hook-job-khdnx        0/1     Completed   0               47m
vss-scheduler-985484c5-9dj2j                            1/1     Running     0               54m
wcm-server-7d4449658-dd8ln                              1/1     Running     0               36m
```

# How to restart

If you've messed with TP too much and want a do-over, run the command below, wait a few dozen minutes, and you get the old TP deleted and a fresh TP delivered:

```
kubectl delete apps sm -n tanzusm
```

I'll write about setup in the next blog post.

