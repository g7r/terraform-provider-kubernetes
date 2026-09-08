# Agent notes for g7r/terraform-provider-kubernetes

Fork of [hashicorp/terraform-provider-kubernetes](https://github.com/hashicorp/terraform-provider-kubernetes),
published to the Terraform Registry as `g7r/kubernetes`. Upstream is the source
of truth for everything except the patch stack listed below.

## Branch model: merge upstream tags, never rewrite

- `doppelganger` is the default and the only working branch. It is upstream's
  release history plus this fork's commits, joined by merge commits.
- Upstream is brought in by **merging its release tags** (`git merge v3.3.0`),
  never its `main`, never by rebase. History on `doppelganger` is never
  rewritten and never force-pushed; every released tag stays reachable from it.
- `main` is a plain mirror of upstream `main` for convenience
  (`gh repo sync g7r/terraform-provider-kubernetes` keeps it fresh). Do not
  commit to it and do not merge it anywhere.
- Fork changes land on `doppelganger` directly or via a feature branch merged
  into it. Do not `gh repo sync` or "Sync fork" into `doppelganger`: GitHub
  would merge upstream `main`, which breaks the tag-based versioning.
- What the fork changes relative to upstream: `git diff v3.2.1 doppelganger`
  (replace with the last merged upstream tag). Fork-only commits:
  `git log --first-parent v3.2.1..doppelganger`.

## Versioning

`<upstream version>-joom.<N>`: `3.2.1-joom.1` is upstream v3.2.1 plus the patch
stack; `N` increments for further fork releases on the same upstream base and
resets to 1 after merging a new upstream tag. The version comes from the git
tag (`v3.2.1-joom.1`), `version/VERSION` is upstream's and unused here.
Terraform requires an exact `version = "3.2.1-joom.1"` pin for prerelease
versions, which suits the consumers (Terragrunt pins exact provider versions).

## Updating to a new upstream release

```sh
git switch doppelganger && git pull --ff-only
git fetch origin --tags            # origin = hashicorp/terraform-provider-kubernetes
git merge v3.3.0                   # the release tag, not main
# Upstream keeps touching the CI files this fork deleted; they come back as
# modify/delete conflicts. Delete them again:
git status --porcelain | awk '$1 == "DU" { print $2 }' | xargs -r git rm -q
# Resolve any remaining conflicts in the patch stack, then:
go build ./... && go vet ./manifest/... && go test ./manifest/...
git commit                         # keep git's "Merge tag 'v3.3.0'" subject
git push
```

Check afterwards that the patch stack still applies semantically: the merge can
succeed textually while upstream restructured the surrounding code. The env
knobs below are the quick smoke test (`K8S_PROVIDER_TYPE_CACHE=1` must still
change plan time on a stack with many `kubernetes_manifest` resources).

## Releasing

```sh
git tag v3.3.0-joom.1 doppelganger
git push g7r v3.3.0-joom.1        # g7r = this fork; adjust to your remote name
```

`.github/workflows/release.yml` runs goreleaser (`.goreleaser.yml`): builds the
same target matrix as upstream, signs `SHA256SUMS` with the GPG key from the
`GPG_PRIVATE_KEY`/`PASSPHRASE` repository secrets, and publishes a GitHub
release with `terraform-provider-kubernetes_<ver>_manifest.json`. The Terraform
Registry ingests it from there within minutes; the public key is registered for
the `g7r` namespace. `snapshot.yml` builds unsigned artifacts on every push to
`doppelganger` and on pull requests as a build check.

Validate locally before tagging when the goreleaser config changed:
`goreleaser check && goreleaser release --snapshot --clean --skip=sign,publish`.

## Patch stack

All behaviour changes are opt-in via environment variables so that a fork build
without them behaves exactly like upstream. Consumers enable them in the
process environment of Terraform (Terragrunt/Atlantis).

| Commit | Knob | Why |
|---|---|---|
| Allow overriding Kubernetes client QPS and burst | `K8S_PROVIDER_QPS`, `K8S_PROVIDER_BURST` | client-go defaults (5 QPS / burst 10) throttle `kubernetes_manifest` refresh to 5 GET/s; 550 manifests = 110 s of waiting. |
| Memoize kubernetes_manifest type derivation per GVK | `K8S_PROVIDER_TYPE_CACHE=1` | `TFTypeFromOpenAPI` re-parses the CRD schema three times per resource per plan. |
| Skip schema hashing while the type cache is disabled | `K8S_PROVIDER_KEEP_HASH=1` restores upstream | `hashstructure.Hash` fed a cache upstream disabled in v2.8.0; 47 % of provider CPU. |
| Add optional pprof HTTP endpoint | `K8S_PROVIDER_PPROF_ADDR=127.0.0.1:6099` | Profile the provider while Terraform drives it. |
| Point the provider address at the g7r fork | — | `registry.terraform.io/g7r/kubernetes` in `main.go` and `manifest/provider/plugin.go`. |
| Drop upstream CI workflows and CRT release config; Release with goreleaser from git tags | — | Upstream releases through HashiCorp's private CRT pipeline. |

Measured on a Terragrunt stack with 550 `kubernetes_manifest` resources:
plan 161 s on upstream 3.2.1, 33 s with QPS=100/Burst=200 and the type cache.

Candidates for upstream: the QPS/Burst override and the type cache. Send them
as separate PRs cherry-picked onto a branch from upstream `main`.

## Boundaries

- Do not edit `docs/`, `templates/`, `_examples/`, `examples/`, `CHANGELOG.md`
  or `.changelog/`: upstream owns them and every edit is a future merge
  conflict. The fork's user-facing notes live in `README.md` ("About this fork").
- Do not rename the Go module path (`github.com/hashicorp/terraform-provider-kubernetes`).
  The registry only needs the provider address, and renaming touches 70 files.
- Do not re-enable the upstream type cache in `manifest/openapi/schema.go`;
  upstream disabled it over a corruption suspicion. The fork caches one level
  up, per GVK.
- Local testing against a real stack: build with `go build -o <dir>/terraform-provider-kubernetes .`
  and point a Terraform CLI config at it via
  `provider_installation { dev_overrides { "hashicorp/kubernetes" = "<dir>" } }`
  (or `g7r/kubernetes` once consumers have migrated). Terraform then skips the
  registry for that provider. Never run apply against production while testing.
