# Secrets

A consuming repository stores no secret at all. Every credential a Deploy
or a Publish needs is the job's own GitHub OIDC token, exchanged at each service that
accepts it; the calling job only grants
`permissions: { contents: read, id-token: write }`.

| Service       | What the token is exchanged for                                          | Where                                                  |
| ------------- | ------------------------------------------------------------------------- | ------------------------------------------------------ |
| Infisical     | The registry account, `DOCKER_REGISTRY_USERNAME` and `DOCKER_REGISTRY_PASSWORD`, read from project `studio-ci`, environment `prod`, path `/` | `connect-infra`, machine identity with OIDC auth       |
| Tailscale     | A node key tagged `tag:ci`, through a trust credential                   | `connect-infra`                                        |
| The cluster   | The user `github:jterrazz/<repo>`, audience `studio-infrastructure`      | `connect-cluster`, an exec plugin minting fresh tokens |

The identifiers in the actions' inputs (Infisical identity id, Tailscale client
ids and audiences) are not secrets. Infisical, Tailscale and the apiserver
each trust a `jterrazz` workflow on `main` or on a `v*` tag, the ref a
`deploy: release` environment ships from: Infisical's identity through one
subject pattern, Tailscale through two trust credentials, one per ref, since a
credential matches one subject (the comments in
`actions/connect-infra/action.yaml` own the exact subjects). A pull request's
run, or a branch's, is trusted by none of them, so it cannot deploy.

The Deploy job declares a GitHub environment, which turns its token's subject
into `repo:…:environment:<env>`. Tailscale trusts it through a third
credential, for every environment; the environment's branch policy is what
limits it to `main` and `v*`. Infisical does not trust it, so that job holds
no registry account: Build stages the chart and the digest for it. The
cluster reads the token's `ref` and owner, not its subject, so it admits the
job as before.

`registry-credentials`, the pull secret a pod mounts, is never handed to the
chart: the Infisical operator syncs it into the namespace. What an app's own
runtime secrets are, and where they live (`/<app>` in project
`studio-infrastructure`), is `studio-infrastructure`'s deploy contract.

`publish-npm.yaml` hands `npm publish` no token either: its publish job grants
`id-token: write`, which provenance signs with and npm's trusted publishing
authenticates with.
