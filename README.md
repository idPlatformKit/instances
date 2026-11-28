# IdPlatformKit instances

**The claims of [IdPlatformKit](https://github.com/idPlatformKit)**: every `Tenant` and `PostgresDatabase` that should exist on the platform, declared in Git.

Argo CD syncs this repository onto the cluster, and the Crossplane compositions from [Orchestrator](https://github.com/idPlatformKit/Orchestrator) turn each claim into real resources. Changing the platform means opening a pull request here, never running `kubectl apply`.

```mermaid
flowchart LR
    PR[Pull request adding a claim] --> CI[CI: claims validated against the XRDs]
    CI --> M[Merge to main]
    M --> A[Argo CD syncs]
    A --> X[Crossplane provisions]
```

## Layout

```
tenants/<tenant>.yaml                   # cluster-scoped: one per team
databases/<namespace>/<name>.yaml       # namespaced: lives in one of the tenant's namespaces, <tenant>-<environment>
apps/<namespace>/<name>.yaml            # namespaced: an app in one environment, written by the app's CI
```

| Kind | API | Docs |
|---|---|---|
| `Tenant` | `platform.idplatformkit.io/v1alpha1` | [tenant.md](https://github.com/idPlatformKit/Orchestrator/blob/main/docs/compositions/tenant.md) |
| `PostgresDatabase` | `database.idplatformkit.io/v1alpha1` | [postgres-database.md](https://github.com/idPlatformKit/Orchestrator/blob/main/docs/compositions/postgres-database.md) |
| `App` | `platform.idplatformkit.io/v1alpha1` | [app.md](https://github.com/idPlatformKit/Orchestrator/blob/main/docs/compositions/app.md). Written by each app's CI ([app-template-go](https://github.com/idPlatformKit/app-template-go)), not by hand |

## Adding a claim

1. Add a file following the layout above. `tenants/checkout.yaml` and `databases/checkout-staging/checkout-db.yaml` are working examples.
2. Open a pull request. CI validates every claim against the Orchestrator XRDs with `crossplane resource validate`. A misspelled kind, a wrong type, a missing required field or an unknown field fails the check.
3. Once merged, Argo CD applies it within a few minutes.

To check a change locally (requires the [Crossplane CLI](https://docs.crossplane.io/latest/cli/) and an Orchestrator checkout next to this one):

```sh
crossplane resource validate --error-on-missing-schemas ../Orchestrator/compositions/ tenants/,databases/,apps/
```

## Deleting is real

Argo CD syncs this repository with `prune` enabled. **Removing a file deletes the resource**: removing a `Tenant` deletes its namespaces and everything in them, including its OpenBao namespace and secrets. Review deletions accordingly.

Databases are the exception, because they hold data:

- Removing a `PostgresDatabase`, or the tenant it lives in, **keeps the database** on the server, unmanaged: adding the same file back adopts it again, data included. To drop it, set `spec.deletionPolicy: Delete` in one pull request, then remove the file in the next.
- An `App` can't be removed until its services are confirmed in its annotation `platform.idplatformkit.io/delete-services` (Argo CD shows the refusal); a service removed from it is kept until confirmed there too. See [Protecting the data](https://github.com/idPlatformKit/Orchestrator/blob/main/docs/compositions/app.md#protecting-the-data).

## Status

Part of the GitOps flow described in [Orchestrator's docs/gitops.md](https://github.com/idPlatformKit/Orchestrator/blob/main/docs/gitops.md). The backing services are dev-mode, single-replica with fixed credentials: this is a reference implementation, not production.

## License

[Apache 2.0](LICENSE)
