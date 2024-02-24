# Sentry helm charts

Sentry is a cross-platform crash reporting and aggregation platform.

This repository aims to support Sentry >=10 and move out from the deprecated Helm charts official repo.

Big thanks to the maintainers of the [deprecated chart](https://github.com/helm/charts/tree/master/stable/sentry). This work has been partly inspired by it.

## How this chart works

`helm repo add sentry https://sync667.github.io/charts`

## Values

For now the full list of values is not documented but you can get inspired by the values.yaml specific to each directory.

## NGINX and/or Ingress

By default, NGINX is enabled to allow sending the incoming requests to [Sentry Relay](https://getsentry.github.io/relay/) or the Django backend depending on the path. When Sentry is meant to be exposed outside of the Kubernetes cluster, it is recommended to disable NGINX and let the Ingress do the same. It's recommended to go with the go to Ingress Controller, [NGINX Ingress](https://kubernetes.github.io/ingress-nginx/) but others should work as well.

Note: if you are using NGINX Ingress, please set this annotation on your ingress : nginx.ingress.kubernetes.io/use-regex: "true".
If you are using `additionalHostNames` the `nginx.ingress.kubernetes.io/upstream-vhost` annotation might also come in handy.
It sets the `Host` header to the value you provide to avoid CSRF issues.

### Letsencrypt on NGINX Ingress Controller
```
nginx:
  ingress:
    annotations:
      cert-manager.io/cluster-issuer: "letsencrypt-prod"
    enabled: true
    hostname: fqdn
    ingressClassName: "nginx"
    tls: true
```

## Clickhouse warning

Snuba only supports a UTC timezone for Clickhouse. Please keep the initial value!

## Upgrading from 3.1.0 version of this Chart to 4.0.0

Following Helm Chart best practices the new version introduces some breaking changes, all configuration for external
resources moved to separate config branches: `externalClickhouse`, `externalKafka`, `externalRedis`, `externalPostgresql`.

Here is a mapping table of old values and new values:

| Before                          | After                         |
| ------------------------------- | ----------------------------- |
| `postgresql.postgresqlHost`     | `externalPostgresql.host`     |
| `postgresql.postgresqlPort`     | `externalPostgresql.port`     |
| `postgresql.postgresqlUsername` | `externalPostgresql.username` |
| `postgresql.postgresqlPassword` | `externalPostgresql.password` |
| `postgresql.postgresqlDatabase` | `externalPostgresql.database` |
| `postgresql.postgresSslMode`    | `externalPostgresql.sslMode`  |
| `redis.host`                    | `externalRedis.host`          |
| `redis.port`                    | `externalRedis.port`          |
| `redis.password`                | `externalRedis.password`      |

## Upgrading from deprecated 9.0 -> 10.0 Chart

As this chart runs in helm 3 and also tries its best to follow on from the original Sentry chart. There are some steps that needs to be taken in order to correctly upgrade.

From the previous upgrade, make sure to get the following from your previous installation:

- Redis Password (If Redis auth was enabled)
- PostgreSQL Password
  Both should be in the `secrets` of your original 9.0 release. Make a note of both of these values.

#### Upgrade Steps

Due to an issue where transferring from Helm 2 to 3. Statefulsets that use the following: `heritage: {{ .Release.Service }}` in the metadata field will error out with a `Forbidden` error during the upgrade. The only workaround is to delete the existing statefulsets (Don't worry, PVC will be retained):

> `kubectl delete --all sts -n <Sentry Namespace>`

Once the statefulsets are deleted. Next steps is to convert the helm release from version 2 to 3 using the helm 3 plugin:

> `helm3 2to3 convert <Sentry Release Name>`

Finally, it's just a case of upgrading and ensuring the correct params are used:

If Redis auth enabled:

> `helm upgrade -n <Sentry namespace> <Sentry Release> . --set redis.usePassword=true --set redis.password=<Redis Password> --set postgresql.postgresqlPassword=<Postgresql Password>`

If Redis auth is disabled:

> `helm upgrade -n <Sentry namespace> <Sentry Release> . --set postgresql.postgresqlPassword=<Postgresql Password>`

Please also follow the steps for Major version 3 to 4 migration

## PostgreSQL

By default, PostgreSQL is installed as part of the chart. To use an external PostgreSQL server set `postgresql.enabled` to `false` and then set `postgresql.postgresHost` and `postgresql.postgresqlPassword`. The other options (`postgresql.postgresqlDatabase`, `postgresql.postgresqlUsername` and `postgresql.postgresqlPort`) may also want changing from their default values.

To avoid issues when upgrade this chart, provide `postgresql.postgresqlPassword` for subsequent upgrades. This is due to an issue in the PostgreSQL chart where password will be overwritten with randomly generated passwords otherwise. See https://github.com/helm/charts/tree/master/stable/postgresql#upgrade for more detail.

## Persistence

This chart is capable of mounting the sentry-data PV in the Sentry worker and cron pods. This feature is disabled by default, but is needed for some advanced features such as private sourcemaps.

You may enable mounting of the sentry-data PV across worker and cron pods by changing filestore.filesystem.persistence.persistentWorkers to true. If you plan on deploying Sentry containers across multiple nodes, you may need to change your PVC's access mode to ReadWriteMany and check that your PV supports mounting across multiple nodes.

## Roadmap

- [x] Lint in Pull requests
- [x] Public availability through Github Pages
- [x] Automatic deployment through Github Actions
- [ ] Symbolicator deployment
- [x] Testing the chart in a production environment
- [ ] Improving the README
