# Upgrading CNPG Operator

based on the documentation https://cloudnative-pg.io/docs/1.28/operator_capability_levels#operator-upgrade, upgrading CNPG operator will trigger rollout restart for all databases created as CNPG is using instance manager that required to be updated https://cloudnative-pg.io/docs/1.28/instance_manager

I use CNPG operator installed with helm chart. check the appVersion to actually know what CNPG operator version you are using https://github.com/cloudnative-pg/charts/blob/main/charts/cloudnative-pg/Chart.yaml

### Upgrading CNPG from 1.26.0 to 1.26.1

1. validating the version

In this case, I upgrade the helm version from 0.24.0 to 0.25.0. Before upgrade, the version is 1.26.0
```
➜  cnpg-learning git:(main) ✗ k get pod cnpg-cloudnative-pg-7bf9fd4554-gdpzx -o yaml | grep image
    image: ghcr.io/cloudnative-pg/cloudnative-pg:1.26.0
    imagePullPolicy: IfNotPresent
    image: ghcr.io/cloudnative-pg/cloudnative-pg:1.26.0
    imageID: ghcr.io/cloudnative-pg/cloudnative-pg@sha256:927d7a8a1f32fe4c1e19665dc36d988f26207d7b7fce81b5e5af2ee0cd18aeef
```

I have 1 postgresql cluster that is properly running
```
➜  cnpg-learning git:(main) k get cluster
NAME              AGE   INSTANCES   READY   STATUS                     PRIMARY
echo-postgresql   33d   3           3       Cluster in healthy state   echo-postgresql-3


➜  cnpg-learning git:(main) ✗ k get pod | grep echo-postgresql
echo-postgresql-2                               2/2     Running   0                5d22h
echo-postgresql-3                               2/2     Running   0                33d
echo-postgresql-4                               2/2     Running   0                8m30s

➜  cnpg-learning git:(main) ✗ k get pod echo-postgresql-3 -oyaml | grep image
    image: ghcr.io/cloudnative-pg/postgresql:17.5
    imagePullPolicy: IfNotPresent
    image: ghcr.io/cloudnative-pg/cloudnative-pg:1.26.0
    imagePullPolicy: IfNotPresent
    image: ghcr.io/cloudnative-pg/plugin-barman-cloud-sidecar-testing:main
    imagePullPolicy: IfNotPresent

    image: ghcr.io/cloudnative-pg/postgresql:17.5
    imageID: ghcr.io/cloudnative-pg/postgresql@sha256:b1deeed2aa998b2f381e39c5cadb9ec06127708c8bd62965743af19abf21628f
    image: ghcr.io/cloudnative-pg/cloudnative-pg:1.26.0
    imageID: ghcr.io/cloudnative-pg/cloudnative-pg@sha256:927d7a8a1f32fe4c1e19665dc36d988f26207d7b7fce81b5e5af2ee0cd18aeef
    image: ghcr.io/cloudnative-pg/plugin-barman-cloud-sidecar-testing:main
    imageID: ghcr.io/cloudnative-pg/plugin-barman-cloud-sidecar-testing@sha256:45bc48d8cc4dc34b81cba9067d356ea60cffc9b2e7946cb073a8c944c84e6b81
```

2. Trigger the operator upgrade

after the operator is upgraded, it will automatically begin to restart the postgresql clusters
```
➜  cnpg-learning git:(main) ✗ k get pod cnpg-cloudnative-pg-7c56b746cb-89dd4 -oyaml | grep image
    image: ghcr.io/cloudnative-pg/cloudnative-pg:1.26.1
    imagePullPolicy: IfNotPresent
    image: ghcr.io/cloudnative-pg/cloudnative-pg:1.26.1
    imageID: ghcr.io/cloudnative-pg/cloudnative-pg@sha256:dfb65c0b6b14e6fb10d36a4e46c0368c062adf5f436c7b0ff0251d408cc34c07
```

it will start to upgrade from the replicas
```
➜  cnpg-learning git:(main) ✗ k get pod | grep echo-postgresql
echo-postgresql-2                               0/2     Init:0/2   0             15s
echo-postgresql-3                               2/2     Running    0             33d
echo-postgresql-4                               2/2     Running    0             2m2s

➜  cnpg-learning git:(main) ✗ k get clusters
NAME              AGE   INSTANCES   READY   STATUS                                       PRIMARY
echo-postgresql   33d   3           2       Waiting for the instances to become active   echo-postgresql-3
```

after replicas upgrade is done, CNPG will switchover the master and upgrade the rest.
```
➜  cnpg-learning git:(main) ✗ k get clusters
NAME              AGE   INSTANCES   READY   STATUS                                       PRIMARY
echo-postgresql   33d   3           2       Waiting for the instances to become active   echo-postgresql-2

➜  cnpg-learning git:(main) ✗ k get pod | grep echo-postgresql
echo-postgresql-2                               2/2     Running       0             81s
echo-postgresql-3                               2/2     Terminating   1 (31s ago)   33d
echo-postgresql-4                               2/2     Running       0             3m8s
```

```
➜  cnpg-learning git:(main) ✗ k get pod echo-postgresql-2 -oyaml | grep image
    image: ghcr.io/cloudnative-pg/postgresql:17.5
    imagePullPolicy: IfNotPresent
    image: ghcr.io/cloudnative-pg/cloudnative-pg:1.26.1
    imagePullPolicy: IfNotPresent
    image: ghcr.io/cloudnative-pg/plugin-barman-cloud-sidecar-testing:main
    imagePullPolicy: IfNotPresent
    image: ghcr.io/cloudnative-pg/postgresql:17.5

    imageID: ghcr.io/cloudnative-pg/postgresql@sha256:b1deeed2aa998b2f381e39c5cadb9ec06127708c8bd62965743af19abf21628f
    image: ghcr.io/cloudnative-pg/cloudnative-pg:1.26.1
    imageID: ghcr.io/cloudnative-pg/cloudnative-pg@sha256:dfb65c0b6b14e6fb10d36a4e46c0368c062adf5f436c7b0ff0251d408cc34c07
    image: ghcr.io/cloudnative-pg/plugin-barman-cloud-sidecar-testing:main
    imageID: ghcr.io/cloudnative-pg/plugin-barman-cloud-sidecar-testing@sha256:fd25a9690ebeab22d09e05f4bd804c78d549a7bda3251b1dd5d42d857a66fdf5
```

there is differences on pod Init Container which using cloudnative-pg:1.26.1, instead of using 1.26.0.