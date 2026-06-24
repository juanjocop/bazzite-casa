# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A [BlueBuild](https://blue-build.org) recipe that bakes a custom Bazzite OCI desktop image,
published to `ghcr.io/juanjocop/bazzite-casa:latest` and signed with cosign. There is **no
application code, no build/test tooling to run locally** — building happens in GitHub Actions
(`.github/workflows/build.yml`), which runs `blue-build/github-action@v1.11` daily at 06:00 UTC,
on every non-`*.md` push, and on `workflow_dispatch`.

The image layers an identity + parental-control stack onto `bazzite-gnome-nvidia-open` so that
the downstream Ansible role `bazzite_ldap` (in the separate `kxs-ansible` repo) only has to
*configure* services, never install them. The role's preflight requires `/usr/sbin/sssd` and
`/usr/sbin/oddjobd` to already exist in the image — that contract drives which packages
`recipes/recipe.yml` installs (sssd/sssd-ldap, oddjob/oddjob-mkhomedir, autofs, timekpr-next).

## Editing the image

Almost all changes happen in `recipes/recipe.yml` (BlueBuild module list: `dnf` install +
`signing`). Static files to copy into the image go under `files/system/` and are referenced from
a `files` module in the recipe. To validate the recipe locally without a full build:

```bash
bluebuild validate recipes/recipe.yml   # if the bluebuild CLI is installed
```

Otherwise rely on the CI build for verification — push to a branch / open a PR and the
`bluebuild` workflow builds the image (PR builds are tagged with the PR number, not pushed to
`:latest`).

## Hard constraints

- **`cosign.key` is the private signing key and is gitignored (`*.key`). Never commit it.**
  Only `cosign.pub` belongs in the repo. The private key lives in GitHub as the `SIGNING_SECRET`
  Actions secret. The key was generated with an empty `COSIGN_PASSWORD`.
- The `signing` module in the recipe embeds the cosign **policy** inside the image. This is why
  the *first* rebase of a host onto this image cannot verify the signature (the policy isn't on
  the host yet) and must use `ostree-unverified-registry:`; subsequent rebases use
  `ostree-image-signed:` and do verify. See README.md "Rebasar el host".
- `base-image: ghcr.io/ublue-os/bazzite-gnome-nvidia-open` with `image-version: stable` must
  match the Bazzite variant actually installed on the target machines.

## Downstream contract

The image ref and match string are consumed by `kxs-ansible`
(`inventory/group_vars/bazzite_desktops.yml`):

```yaml
bazzite_image_ref: "ostree-image-signed:docker://ghcr.io/juanjocop/bazzite-casa:latest"
bazzite_image_match: "bazzite-casa"
```

Changing the image name, registry, or signing scheme breaks that consumer — update both sides.
