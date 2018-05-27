# oidckube

Oidckube functions as a wrapper around [minikube](https://github.com/kubernetes/minikube) that will initialize, deploy,
and partially configure the instance to use [Keycloak](https://www.keycloak.org/); an Open Source Identity and Access
Management tool as an Authentication Source.

## Requirements

* [minikube](https://github.com/kubernetes/minikube)
* [Virtualbox](https://www.virtualbox.org/wiki/Downloads)
* [cfssl and cfssljson](https://pkg.cfssl.org/)
* [jq](https://stedolan.github.io/jq/download/)

**NOTE:** This script only supports Virtualbox as the virtualization provider for minikube. If the cfssl and jq 
requirements are not found, it will attempt to download and install them locally into the `bin` sub-directory.

## Usage

1. Within the project directory, create a `config` file based off the supplied config example (`config.example`). If
you opt to forgo doing so, one based off the `config.example` file will be generated automatically. This file is used
by both `oidckube.sh` and `login.sh` to configure and authenticate to Keycloak.

|         Variable         |       Default       |                                                              Description                                                             |
|:------------------------:|:-------------------:|:------------------------------------------------------------------------------------------------------------------------------------:|
|    `KEYCLOAK_ADDRESS`    | `keycloak.devlocal` | Address for the locally deployed instance of Keycloak                                                                                |
|   `KEYCLOAK_AUTH_REALM`  |        `k8s`        | Name of the realm within Keycloak used for Kubernetes Authentication                                                                 |
|   `KEYCLOAK_CLIENT_ID`   |      `oidckube`     | Name of the OIDC client used for Kubernetes Authentication                                                                           |
| `KEYCLOAK_CLIENT_SECRET` |                     | OIDC Secret associated with the Client ID. **NOTE:** This cannot be populated ahead of time, and is is generated by Keycloak itself. |

2. Run `./oidckube.sh init`. This will automate the certificate generation, CA certificate insertion, deploy
Keycloak, and configure minikube to use the Host's DNS resolver.
3. Modify your system's `/etc/hosts` file with the information printed out from the previous step. This will allow 
both your host and minikube instance to reference the `KEYCLOAK_ADDRESS`.
4. Run `./oidckube.sh start` to start the minikube instance up with the generated OIDC config.
5. Login to keycloak administrator portal by going to `https://<KEYCLOAK_ADDRESS>` e.g. `https://keycloak.devlocal`,
and use the credentials `keycloak` / `keycloak` **NOTE:** Keycloak takes a few moments to start after minikube comes up
and may not be immediately accessible once booted.
6. Create a new auth realm using the same name as defined in the `KEYCLOAK_AUTH_REALM` config. **NOTE:** If you are
using the default config, at this time you may import the `k8s-realm-example.json` to skip the group and client
configuration (you will however have to generate a new client secret). For the import, select only `Import groups`, 
`Import clients`, and `Import client roles`, then set it to `skip` if the resource already exists.
7. Navigate to the `clients` section and create a new client.
8. Give it the same name as defined in the `KEYCLOAK_CLIENT_ID` config.
9. At the new client configuration page, change the `Access Type` to be `confidential`, and configure the
`Valid Redirect URI` to be `https://<KEYCLOAK_ADDRESS>/*`. Then press `Save`.
10. Click on the `Mappers` Tab and then `Create`.
11. Call this new mapping `groups`, set the `Mapper Type` to `Group Membership` and `Token Claim Name` to `groups`,
then save.
12. Add a second Mapping, called `email_verified`. Set the `Mapper Type` to `Hardcoded claim`, the `Token Claim Name`
to `email_verified`, `Claim value` to `true`, and `Claim JSON Type` to `boolean`. This is **ONLY** required in
versions of Kubernetes less than 1.11. For information regarding this claim, see this Github Issue:
[kubernetes/kubernetes#59496](https://github.com/kubernetes/kubernetes/issues/59496).
13. Click on the `Credentials` Tab and generate a new secret, then copy the `Secret` and update the config file setting
`KEYCLOAK_CLIENT_SECRET` to the newly generated value.
14. Navigate to the `Groups` section and create 2 new groups: `cluster-users` and `cluster-admins`. These map to the
cluster role bindings created during initialization (`manifests/crb-users.yaml` and `manifests/crb-admins.yaml`).
15. Goto `Users` and create two new users giving them fake emails e.g. `admin@keycloak.devlocal` and 
`user@keycloak.devlocal`, assigning them a password under the `Credentials` tab, add lastly add one to each of the
groups created in the previous step. At this point, Keycloak is now configured.
16. Run `./login.sh`. It will prompt you for a username and password. Use the email address of one of the accounts
created earlier. the `./login.sh` script will add the user automatically to your kube config.
17. Create a new context using the newly added account. e.g:
```
$ kubectl config set-context oidckube-user --cluster=minikube --user=user@keycloak.devlocal --namespace=default
<or>
$ kubectl config set-context oidckube-admin --cluster=minikube --user=admin@keycloak.devlocal --namespace=default
```
Both the instance of minikube and your local client should be configured to use oidc for server authentication.
The cluster role bindings map the group `cluster-users` to the `view` cluster role, and `cluster-admins` to the
cluster-admin` role.

