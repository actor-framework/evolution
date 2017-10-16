# CAF Evolution

This repository tracks proposals for changes to CAF, the C++ Actor Framework. A
proposal can:

* Change the public API
* Introduce new features
* Improve existing features

Bug reports and small contributions such as bugfixes and low-impact changes
should use CAF's main repository. Either by opening an
[issue](https://github.com/actor-framework/actor-framework/issues) or a
[pull request](https://github.com/actor-framework/actor-framework/pulls).

## Contributing a Proposal

1. Present your idea on CAF's
   [development mailing list](https://groups.google.com/forum/#!forum/caf-devel)
2. Once there is sufficient interest or support for your idea write down a
   proposal, based on the `0000-template.md` file and create a
   [pull request](https://github.com/actor-framework/evolution/pulls). Please
   pick the next unused ID for your document (also look at open PRs!) and add
   your proposal to the `index.xml` file with status `awaiting` in your PR.
3. The pull request is merged once the document clearly outlines the proposed
   solution and its impact on CAF's source code.
4. Announce the proposal on the mailing list and wait for review decisions.
   Note that an accepted proposal does not guarantee an implementation by the
   core team.

## Contributing Implementations for Proposals

1. Please announce that you are working on an implementation for a proposal on
   the development mailing list. This is to make sure no two people are working
   on the same proposal independently, ultimately wasting time and effort.
2. Once you have an implementation available, create a
   [pull request](https://github.com/actor-framework/actor-framework/pulls) in
   the main repository. Please reference the proposal you are implementing.
3. Address any feedback on your PR until it is merged.

# Status of a Proposal

The status of a document is on of the following, starting with awaiting when
created:

* **awaiting:** Awaiting review.
* **active:** Currently under review.
* **accepted:** Accepted RFC that has not been implemented.
* **implemented:** Accepted and implemented in CAF.
* **rejected:** Rejected by the review process.
* **withdrawn:** Withdrawn by the proposer.
