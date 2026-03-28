# SONiC Branch Naming Policy

## Table of Contents

1. [Revision](#revision)
1. [Scope](#scope)
1. [Definitions/Abbreviations](#definitionsabbreviations)
1. [Branch Types](#branch-types)
1. [Summary](#summary)
1. [References](#references)

## Revision

| Rev |     Date    |               Authors                 | Change Description |
|:---:|:-----------:|:-------------------------------------:|--------------------|
| 0.1 | 02/26/2026  |   Justin Sherman, Ko Outlaw-Spruell   | Initial version    |
| 0.2 | 03/05/2026  |   Justin Sherman                      | Addressed comments |
| 0.3 | 03/09/2026  |   Justin Sherman                      | Addressed comments |
| 0.4 | 03/27/2026  |   Justin Sherman                      | Addressed comments |

## Scope

This document establishes a standard naming convention for git branches in all Cisco whitebox repositories. 
Standard naming greatly simplifies branch protection and provides clear indication of the purpose and status of each branch, including what SONiC DevOps support it receives, if any.

## Definitions/Abbreviations

| **Term**                 | **Meaning**                                                                                                                         |
|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| DevOps                   | Development Operations, the team and infrastructure used to support developers throughout the full development and release cycle    |
| Mirror Branch            | A branch used to mirror external sources only                                                                                       |
| Trunk Branch             | Long-lived branch that receives continuous development via pull requests and usually is protected against build and test breakages  |
| Release Throttle Branch  | A branch created for the purposes of a specific release, which accepts few, if any, commits once created                            |
| Branch Protections       | A GitHub feature that establishes protection policies for changes made to specific branches to ensure continuous quality            |
| GitGuardian              | A security tool mandated by Cisco Security & Trust Organization (STO) to prevent committing of secrets (passwords, tokens, etc)     |
| CODEOWNERS               | A GitHub feature which allows developers to set mandatory pull request reviewers based on which files have been modified in the PR  |
| Ring 1/2                 | Cisco SONiC pipelines used to ensure continuous quality on important branches, by validating pull request changes before merge      |
| Ring 3                   | The Cisco SONiC pipeline used to build and test trunk and release throttle branches on a daily basis                                |

## Branch Types

### Mirror Branches

These are pure mirrors—or almost pure mirrors—of upstream git repos. 
They do not accept any local commits and are prefixed with “mirror/” or “azure/cisco” in the case of existing sonic-buildimage mirror branches. 
They are eligible for Ring 3 builds. 
They do not support Ring 1 or 2, as PRs are never raised against them.

One notable exception is the `master` branch of the whitebox/SONiC repo, which receives merges from upstream to enable local changes in the `doc/cisco` directory only.

### Cisco Main Branches

These are trunk branches that receive commits via pull request and are built regularly in Ring 3 streams. 
All merges to these branches must be made via PR, Jira ID must be present in the PR headline, and UT/precommit/Azure pipelines must be run as well. 
GitGuardian checks must succeed on all PRs. 
They are eligible for Ring 3 streams and can be used as direct release branches. 
They are prefixed with “cisco/”. For smaller repos, a single “main” branch is acceptable.

### Release Throttles

These are throttle branches taken off of main trunk branches and intended to receive few—if any—commits once created. 
They provide a manner for taking a snapshot of a release build or release train, while enabling high priority point fixes to be pushed if customers ever require them. 
Changes to them should be very minimal in most cases. 
They have all of the same protections and abilities as trunk branches, and additionally require PR merges to be made by a release manager.

### Development Trunk Branches

These are trunk branches used for projects that are in the earlier stages of development. 
They offer a place for smaller and newer teams to benefit from trunk-based development without the same constraints of larger teams who need to ensure their trunk branches are stable at all times. 
Branches are prefixed with “dev/<PROJECT NAME>/”. 
Pull requests with a valid Jira ID are still required for merge, but PR approval is not required. 
If a CODEOWNERS file is present in the branch, PRs must abide by it. 
Ring 1 and 2 are supported but not required. 
These branches are eligible for Ring 3 builds as a courtesy, but since precommit checks are not enforced, Ring 3 breakages will not be triaged or debugged by the DevOps team. 
If formal support is needed, a main trunk branch must be used instead.

### Ephemeral Branches

All branches which don’t meet the above criteria are considered ephemeral branches. 
These are not subject to any branch protections, may **never** be used in Ring 3 builds or customer releases, and receive no support from the DevOps team. 
These are typically used for short-lived feature development or bug fixes before being merged to an appropriate trunk branch via PR.
These branches typically receive commits from a single user and have an expected lifetime of a sprint. The standardized naming format will be determined in a future document.

## Summary

| **Type**           |  **Use Case**                                                           |  **Prefix(es)**             |	**PR Required** |	**Precommit Build Required** | **Allowed in Ring 3** | **DevOps Support** |
| -------------------|-------------------------------------------------------------------------|-----------------------------|------------------|------------------------------|-----------------------|--------------------|
| Mirror             | Provide a read-only local mirror of upstream code                       | `mirror/` or `azure/cisco/` |                  |                              | Yes                   | Yes                |
| Main Trunk         | Continuous, stable builds for nightlies and customer releases           | `cisco/`                    | Yes              | Yes                          | Yes                   | Yes                |
| Release Throttle   | Snapshot a customer release and enable future point fixes on it         | `release/`                  | Yes              | Yes                          | Yes                   | Yes                |
| Dev Trunk          | Collaborate with a small team during the early stages of a project      | `dev/<PROJECT>/`            | Yes              | No                           | Yes                   | No                 |
| Ephemeral Branches | Used for short-term development (single Jira) before merging into trunk | _To be determined_          | No               | No                           | No                    | No                 |

## References

- [Trunk Based Development: Introducion](https://trunkbaseddevelopment.com/)
- [About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [GitGuardian](https://www.gitguardian.com/)
- [About code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
