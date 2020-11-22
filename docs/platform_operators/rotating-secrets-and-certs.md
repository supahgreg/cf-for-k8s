# Rotating secrets
Most secrets in cf-for-k8s can be rotated by simply changing the values in your `cf-values.yml` file and running a standard deploy using `ytt` and `kapp`. The rotation is complete when the `kapp` deploy succeeds.

## Exceptions

The following fields currently cannot be rotated:

* `blobstore.secret_access_key`
* `capi.database.encryption_key`
* `capi.database.password`

For example, rotating the Cloud Controller DB encryption key is a breaking change. Rotating the key will
require a recreating your environment (including deleting database contents) in order to prevent decryption errors when fetching
previously-saved data.

If you find you must rotate one of the above fields: 

1. Delete your CF installation with `kapp delete -a cf` (assuming a `kapp` app name "cf")
1. Then, in your `cf-values.yml`, you will need to update the properties that need rotating.
1. Enter your updated values and rerun `kapp deploy -a cf ...`. 

## Specific issues

### Outright errors

* [Rotating non-admin user/role credentials in postgres fails](https://github.com/cloudfoundry/cf-for-k8s/issues/216) 

* [Support rotation of `blobstore.secret_access_key`](https://github.com/cloudfoundry/cf-for-k8s/issues/527)

* [Support rotation of `capi.database.encryption_key`](https://github.com/cloudfoundry/cf-for-k8s/issues/529)

* [Support rotation of `capi.database.password`](https://github.com/cloudfoundry/cf-for-k8s/issues/530)

* [Support rotation of `uaa.database.password`](https://github.com/cloudfoundry/cf-for-k8s/issues/566)

### Concourse/CI issues

The following fields can be modified and are updated eventually, but
`uptimer`-type checking in the pipelines are currently configured to
use either the old password or the new one, and will fail.

* `cf_admin_password` - manual upgrade works, CI/upgrade fails

## Notes on currently supported rotations

### Rotating ingress certificates
To rotate the application domain certificate or system domain certificate, you
can do the following:

1. In your `cf-values.yml`, the system domain certificate is called
`system_certificate` and the application domain certificate is called
`workloads_certificate`. These two properties can be independently updated.
2. Redeploy using `ytt` and `kapp`. This will cause the secrets to be recreated and
successfully rotated in the Istio Ingress Gateway.

If you have multiple app domains, they all share the `workloads_certificate`.
You cannot rotate the app domains' certificates separately because there is only
one certificate.
