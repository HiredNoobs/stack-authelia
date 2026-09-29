# stack-authelia

Authelia deployment for k8s, SSO for the services behind nginx.

A single instance StatefulSet runs on a node labelled `authelia_core_authelia` (set in iac-homelab). Its storage is SQLite, so it can't run more than once, if more nodes are labelled it still runs once, on any of them. The data (2FA registrations and the notifier's file) is on a `$AUTHELIA_STORAGE_CLASS` volume, so it survives restarts and follows the pod to another labelled node. Sessions are kept in memory, so a restart logs everyone out.

It needs:

- `stack-longhorn`, for the volume.
- `stack-lldap`, for the users and groups. Authelia checks LDAP when it starts, `deploy` stops if LLDAP isn't ready.

It's reached through nginx as `https://auth.$DOMAIN`. The `authelia` Service (port 9091) has no LoadBalancer IP.

The config is `configs/configuration.yml.tmpl`, `deploy` fills in the stack's variables and stores it in a ConfigMap, changing it restarts the pod.

`clean` keeps the data volume, `purge` removes it and every 2FA registration with it.

## Secrets

`secrets.env` needs:

- `AUTHELIA_JWT_SECRET` - signs identity verification links.
- `AUTHELIA_SESSION_SECRET` - encrypts the sessions, changing it logs everyone out.
- `AUTHELIA_STORAGE_ENCRYPTION_KEY` - encrypts the database, at least 20 characters. **Without it the 2FA registrations can't be read**, so keep it in Vault.
- `AUTHELIA_LDAP_PASSWORD` - the password of the `$AUTHELIA_LDAP_USER` user in LLDAP.

Generate them with `openssl rand -base64 48`, then `deployment secret_push` so they're kept in Vault.

## LLDAP

Authelia binds as `$AUTHELIA_LDAP_USER` (`authelia`), which needs creating in LLDAP (`https://lldap.$DOMAIN`) first:

1. Create the `authelia` user, with a placeholder email (e.g. `authelia@$DOMAIN`) and `AUTHELIA_LDAP_PASSWORD` as its password.
2. Add it to `lldap_strict_readonly`, it can read users and groups but not change them.
3. Create an `admins` group and add yourself to it, see [Access control](#access-control).

As it can't change passwords, Authelia's password reset is turned off. Passwords are changed in LLDAP.

## Access control

The default policy is `deny`, members of `admins` get into every `*.$DOMAIN` service with 2FA. Other groups and per-service rules are added in `configuration.yml.tmpl` as services are put behind Authelia.

Nothing is protected by just deploying Authelia, services are put behind it in `stack-nginx`.

## Registering 2FA

The notifier writes to a file rather than sending emails. When Authelia asks to confirm your identity (e.g. registering a TOTP app or security key), get the code with:

```bash
deployment authelia_notifications
```
