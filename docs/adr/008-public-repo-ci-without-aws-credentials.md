# ADR-008: Public-repo CI without AWS credentials

- Status: Accepted
- Date: 2026-10-10

## Context

Five of the seven repositories are public (`lz-docs`, `lz-governance`, `lz-network-stack`, `aft-global-customizations`, `aft-account-customizations`). Anyone can read them, and pull requests run CI. Storing AWS credentials, or an OIDC role that can change real infrastructure, in a public repository's CI would widen the blast radius of a mistake, a malicious PR or a leaked log. The lab has no credit cushion, so an abused account would cost real money.

## Decision

1. CI in public repositories has **no AWS credentials** and no OIDC role. It only runs checks that need no cloud access: `gitleaks` (full-history secret scan), `terraform fmt`, `tflint`, and static policy checks such as `checkov`.
2. `terraform plan` against real accounts and every `apply` run only from the owner's machine or from the private AFT pipelines. Neither runs in public CI.
3. The default branch of every public repository is protected by a branch ruleset:
   - changes go through a pull request;
   - the status check `gitleaks` (workflow `secret-scan`) must pass before merge;
   - force pushes to the default branch are blocked.
4. GitHub Secret scanning and Push protection are enabled on all public repositories.
5. Commits in public repositories use the GitHub `noreply` author address, not a personal e-mail.

## Why the ruleset is part of this decision

On 2026-10-10 a test pull request that added a fake key made `secret-scan` fail, yet GitHub still showed the PR as mergeable. GitHub does not require checks by default, so detection alone does not prevent a secret from reaching `main`. The ruleset turns detection into prevention.

## Consequences

- Every change to `main` needs a PR, even for a solo maintainer. This is intended practice for the lab.
- Rewriting history (for example fixing an author address) needs the ruleset to be disabled temporarily and enabled again afterwards.
- A required check can only be selected after the workflow has run at least once in that repository.
- The control covers public repositories only. The two private repositories (`aft-account-request`, `aft-deployment`) are outside this ADR.
- Detection is not a fix for a real leak. If a real secret is ever pushed, rotate it first and clean the history second.
