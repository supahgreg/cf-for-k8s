# Setup ingress certs with Let's Encrypt
The following instructions will setup ingress certs with Let's Encrypt. You have the option of setting up certs before install or update an existing install with the new certs.

## Objective
At the end of this setup, you and your users will be able to access CF CLI and CF APPs over HTTPS.

## Prerequisites

- `certbot` cli
   - For Mac, run `brew install certbot`. For Linux distros, see the instructions on the [certbot site](https://certbot.eff.org/instructions) [1].
- Permissions to add/update DNS `A` and `TXT` records.

[1] On the `certbot` site, the web server and OS is irrelevant. You will be generating the certs on your machine, so choose the OS that matches your OS.

## Steps to setup ingress certs
The following instructions assume that the system domain is setup at `pm-k8s.dev.relint.rocks` and the apps domain is setup at `apps.pm-k8s.dev.relint.rocks`. You should update the domains accordingly.

### System domain

1. Set the environment variable for sys domain
```console
export SYS_DOMAIN=pm-k8s.dev.relint.rocks
```
2. Generate a cert for the system domain
```console
certbot --server https://acme-v02.api.letsencrypt.org/directory -d "*.$SYS_DOMAIN" --manual \
    --preferred-challenges dns-01 certonly \
    --work-dir /tmp/certbot/wd --config-dir /tmp/certbot/cfg \
    --logs-dir /tmp/certbot/logs
```
3. You will be presented with a challenge to verify domain ownership. Copy the `TXT` value printed by certbot and create a TXT record in your DNS provider.
```console
# example of the TXT in your DNS
_acme-challenge.SYS_DOMAIN.	TXT    kyfxzsAirB79lsk173jkdlamxiryqloy
```
4. Wait for the TXT record to propagate to the nameservers. You can `dig` tool in a separate console to verify the TXT is updated
```console
dig _acme-challenge.$SYS_DOMAIN TXT
```
5. In the certbot console, press enter once the TXT change is propagated to nameservers. `certbot` will verify that you own the server and create the necessary files.

### Apps domain
Let's now create apps domain cert.

1. Set environment variable for the apps domain
```console
export APPS_DOMAIN=apps.pm-k8s.dev.relint.rocks
```
2. Generate a cert for the workloads domain
```console
certbot --server https://acme-v02.api.letsencrypt.org/directory -d "*.$APPS_DOMAIN" --manual \
    --preferred-challenges dns-01 certonly \
    --work-dir /tmp/certbot/wd --config-dir /tmp/certbot/cfg \
    --logs-dir /tmp/certbot/logs
```
3. You will be presented with a challenge to verify domain ownership. Copy the `TXT` value printed by certbot and create a TXT record in your DNS provider.
```console
# example of the TXT in your DNS
_acme-challenge.$APPS_DOMAIN.	TXT    kyfxzsAirB79lsk173jkdlamxiryqloy
```

### Update cf-values YAML
The following instructions assume you have created `cf-values.yml`. Please ensure to copy the file contents into the variables as is.

1. **Update system certificate values**

    Lookup `system_certificate` in `cf-values.yml`. You should config variables `crt`, `key` and `ca`. Follow the instructions below,
    ```yaml
    system_certificate:
      crt: |
        <replace this with the contents of the file /tmp/certbot/cfg/live/$SYS_DOMAIN/fullchain.pem>
      key: |
        <replace this with the contents of the file /tmp/certbot/cfg/live/$SYS_DOMAIN/privkey.pem>
      ca: "" #! replace whatever old value with empty string
    ```
    Your final output for `system_certificate` should look something like
    ```yaml
    system_certificate:
      crt: |
        -----EXAMPLE CERTIFICATE-----
        ...
      key: |
        -----EXAMPLE RSA PRIVATE KEY-----
        ...
      ca: ""
    ```

1. **Update apps certificate values**

   The `workloads_certificate` has sub-keys `crt`, `key`, `ca` under it.
   ```yaml
   workloads_certificate:
      crt: |
        <replace this with the contents of the file /tmp/certbot/cfg/live/$APPS_DOMAIN/fullchain.pem>
      key: |
        <replace this with the contents of the file /tmp/certbot/cfg/live/$APPS_DOMAIN/privkey.pem>
      ca: "" #! replace whatever old value with empty string
   ```

1. Follow the instructions in [the deploy doc](../deploy.md) to generate the final deployment YAML (using `ytt`) and use `kapp` to deploy cf-for-k8s to your cluster.

### Verify TLS

1. Verify your sys domain cert by connecting to the CF API without skipping SSL validation or reviewing the cert in a browser.
    ```console
    cf api https://api.$SYS_DOMAIN
    ```

1. Follow the instructions in [the deploy doc](../deploy.md) to set up and org and space, and `cf push` an app (if you haven't already).


1. Verify your app domain cert by running `curl -vvv` or reviewing the cert in a browser.

```console
curl -vvv https://$APP_NAME.$APPS_DOMAIN
# output should show `SSL certificate verify ok`
```
