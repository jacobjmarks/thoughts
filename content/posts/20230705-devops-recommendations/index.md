---
layout: post
title: DevOps Recommendations
date: 2023-07-05T00:00:00.000+1000
tags: [CI/CD, DevEx, DevOps, Azure DevOps]
summary: "Recommendations for the configuration, management and utilisation of DevOps tooling when working on a project."
cover:
  image: "cover.png"
  relative: true
ShowToc: true
---

The following post provides recommendations for the configuration, management and utilisation of DevOps tooling when working on a project.

While primarily in the context of Azure DevOps, concepts are written to be largely agnostic of the platform of choice and these recommendations are transferable.

> Be mindful of any existing ways of working and processes that may be in place when joining ongoing projects. You should consider these recommendations in high regard but diverge from them whenever and wherever necessary to suit the needs of your project and team.

## Iterations

Iterations should be configured to improve Work Item tracking and traceability. Work Items should be assigned to a given Iteration to indicate when the work is planned to be completed.

An Iteration most simply represents a period of time and will usually match a given Sprint.

The following template is recommended when creating Iterations:\
`{Project}/{Phase}/{Iteration}`

For example:\
`Awesome Project/Discovery/Sprint 01`

> If your DevOps platform does not support Iteration nesting &mdash; as recommended above &mdash; use your best judgment and the tools at your disposal to implement a strategy that suits your needs.

## Work Items

Work Items should be created to represent any pieces of work that need to be completed. They should be created before the work they represent has started and should be assigned to an Iteration.

Work Items should always contain a detailed description as well as a set of acceptance criteria that can be used to definitively assert whether the work is completed or not. This is especially important as Work Items may be reviewed and/or completed by individuals other than the author.

## Visibility

As work is planned and completed, it is important to ensure Work Items are kept up to date concerning their status and assignee. At a minimum the status may include *To Do*, *Doing*, and *Done*, or equivalent. Ensuring Work Items are assigned a status that accurately reflects the state the Work Item is in, as well as ensuring the individual who is actioning that piece of work has been appropriately assigned, is imperative in providing visibility across the project in regards to what is being done and who is doing it. This information is especially crucial for the leads on the project to ensure they understand the state of affairs.

## Traceability

### Linking Work Items

Work Items should be linked to other Work Items where there exists a relationship. This relationship can be direct or indirect and represent for example a dependency, blocker, duplicate, hierarchy, etc. Accurately representing these relationships is extremely useful in supporting traceability and providing additional meaning to Work Items in the context of others.

### Linking Development

Code commits and Pull Requests should always be linked to the Work Item to which the work relates. This is imperative for maintaining traceability throughout the project and associating the *what* with the *why*. Not only can contributions be viewed from the Work Item &mdash; providing evidence and detail of work &mdash; but perhaps more importantly, Work Items can be viewed from contributions &mdash; providing the reasoning behind why the changes were (or are being) made. This is another reason why Work Items should have comprehensive descriptions.

## Policies

Policies should always be configured for each code repository on at least the default branch (typically `main`).

The following configurations are recommended.

> Note that configuring at least one required policy will protect the target branch, enforcing changes be made via Pull Requests and preventing the branch from being deleted. This is critically important to ensure all processes are considered and the target branch &mdash; as your source of truth &mdash; is protected.

### Branch Policies

[Branch policies and settings | Microsoft Learn](https://learn.microsoft.com/en-au/azure/devops/repos/git/branch-policies?view=azure-devops&tabs=browser)

#### Require a minimum number of reviewers

*Minimum number of reviewers: 1*\
*Allow requestors to approve their own changes: Yes*

Contributions should typically always be reviewed by at least one individual other than the author. However, strictly enforcing this requirement can cause unnecessary friction in certain circumstances. The onus should instead be placed on the author to seek an appropriate level of external review for the contribution which is being made. The author should take ownership of their contribution and be respectful of others' time, not overloading an individual with a large number of trivial changes which they must review. Configuring this policy in this manner will require a "tick of approval" for all Pull Requests however allow the author to provide it if external review is not necessary.

#### Check for linked work items

*Optional*

While Work Items should always be linked to Pull Requests, there are some scenarios in which contributions may be made outside the context of a tracked Work Item. While not ideal &mdash; as all work *should* be tracked &mdash; this scenario should be supported without the need for invasive Branch Policy bypassing. Enabling this policy will encourage and remind Pull Request authors to link associated Work Items, yet allow the policy to be considered optional if ultimately necessary.

#### Check for comment resolution

*Required*

All comments which are made against the Pull Request should be resolved before merging. This configuration is especially useful to prevent the scenario in which feedback is given on a Pull Request and the Pull Request is merged &mdash; either manually or automatically &mdash; before the feedback is actioned.

#### Limit merge types

*Allowed merge types: Squash merge*

While perhaps slightly more subject to the preferences of the team, limiting the available merge types can enforce a level of consistency against the git history of the target branch. Maintaining a clean, concise and readable git history might not generally be high on the list of project goals and priorities however it is a luxury that can pay dividends. It makes it easier to decipher the order in which commits were made; simplifies automated changelogs and/or release notes; and facilities easy rollbacks if particular contributions are found to be erroneous. Utilising the *Squash merge* for all Pull Requests &mdash; such that all interim commits are "squashed" into a single commit when merged &mdash; is a good way of maintaining a high-quality git history.

See [Merge strategies and squash merge | Microsoft Learn](https://learn.microsoft.com/en-au/azure/devops/repos/git/merging-with-squash?view=azure-devops)

### Build Policies

[Branch policies and settings - Build Policies | Microsoft Learn](https://learn.microsoft.com/en-au/azure/devops/repos/git/branch-policies?view=azure-devops&tabs=browser#build-validation)

There should be at least one *Required* build policy configured against the default branch. This policy should at a minimum both build the application and run the application's test suite/s. This is imperative to assert that new changes which are to be merged will break neither the build nor any of the existing tests. It asserts a minimum level of quality for any contributions and, in this respect, gives reviewers some piece of mind.

## Summary

✅ **Define Iterations for the project**

✅ **Ensure Work Items have a detailed description and acceptance criteria**\
Assign them to an appropriate Iteration

✅ **Ensure relationships between Work Items are represented**

✅ **Configure Branch Policies for each repository's default branch**\
Require a minimum number of reviewers\
*Minimum number of reviewers: 1*\
*Allow requestors to approve their own changes: Yes*

Check for linked work items\
*Optional*

Check for comment resolution\
*Required*

Limit merge types\
*Allowed merge types: Squash merge*

✅ **Configure a Build Policy for each repository's default branch**\
  Build and Test the application at a minimum

✅ **Ensure Work Items are linked to code contributions**

✅ **Ensure the Work Item state is kept up to date as progress is made**

\
~ These were my thoughts.
