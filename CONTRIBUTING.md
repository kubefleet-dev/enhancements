# Contributing

KubeFleet welcomes contributions and suggestions!

## First things first

### Terms

All contributions to the repository must be submitted under the terms of the [Apache Public License 2.0](https://www.apache.org/licenses/LICENSE-2.0).

### Certificate of Origin

By contributing to this project, you agree to the Developer Certificate of Origin (DCO). This document was created by the Linux Kernel community and is a simple statement that you, as a contributor, have the legal right to make the contribution. See the [DCO](DCO) file for details.

### DCO Sign Off

You must sign off your commit to state that you certify the [DCO](DCO). To certify your commit for DCO, add a line like the following at the end of your commit message:

```
Signed-off-by: John Smith <john@example.com>
```

This can be done with the `--signoff` option to `git commit`. See the [Git documentation](https://git-scm.com/docs/git-commit#Documentation/git-commit.txt--s) for details.

### Code of Conduct

The KubeFleet project has adopted the CNCF Code of Conduct. Refer to our
[Community Code of Conduct](https://github.com/kubefleet-dev/kubefleet/blob/main/CODE_OF_CONDUCT.md) for details.

## KubeFleet enhancement proposal (FEP) purpose and guidelines

A KubeFleet enhancement proposal (FEP) is a design document that describes a new feature, experience, or an improvement
to existing features/experiences in KubeFleet. The document features a detailed technical specification of
the feature/experience and a rationale for it.

The proposals help KubeFleet contributors and maintainers to propose new features/experiences, to collect feedback
from the community, and to document any decisions made on the development of KubeFleet.

Each proposal is uniquely identified by an ordinal number, zero-padded to 4 digits (e.g., `0001`, `0042`).

### The workflow

* Start with an idea for KubeFleet.
    * Anyone can write a KubeFleet enhancement proposal. Please refer to the [Is my thing an enhancement?](README.md#is-my-thing-an-enhancement) section to check if a KubeFleet enhancement proposal is needed for your idea.
    * To avoid unnecessary back-and-forths, it is recommended that you discuss your idea with the community first
    to gather initial feedback and support, validate the concept, and identify stakeholders who would be interested
    in implementing and maintaining the enhancement.
* Submit a proposal.
    * Create an issue in the repository to track the proposal. The issue should be of the title format
    `FEP-[NNNN]: [YOUR-PROPOSAL-TITLE]`, where `NNNN` is the next available ordinal number not used by any published
    proposal or proposal that has been submitted for review, and `YOUR-PROPOSAL-TITLE` is the title of your proposal.
    * Fork the repository, and create a new directory for your enhancement proposal under the path `enhancements/`.
    The name of the directory should be of the format `[NNNN]-[short-description-of-the-proposal]`, where `NNNN` is
    the ordinal number of your proposal, and `short-description-of-the-proposal` is a short description of the
    proposal in lowercase letters and the kebab case format. For example, the directory might be named
    `0001-placement-policy-api`.
    * Write the proposal in a file named `README.md` under the proposal directory. Follow the structure and directives
    as given by the template provided in [NNNN-FEP-template/README.md](enhancements/NNNN-FEP-template/README.md). You can
    add additional files to the proposal directory as needed, such as diagrams, code snippets, and other supporting
    materials.
    * Send a pull request to the `main` branch of this repository with your proposal directory. The PR should be of the
    title format `FEP-[NNNN]: [YOUR-PROPOSAL-TITLE]`, where `NNNN` is the ordinal number of your proposal, and
    `YOUR-PROPOSAL-TITLE` is the title of your proposal. For example, the PR might be titled `FEP-0001: Placement Policy API`.
    Link the PR to the issue created earlier.
* Discuss and iterate on the proposal.
    * The proposal will be reviewed by the [KubeFleet maintainers](https://github.com/kubefleet-dev/kubefleet/blob/main/MAINTAINERS.md),
    but anyone in the community is welcome to provide feedback and suggestions.
    * Discussions can happen in all available channels; however, make sure that all discussions are eventually captured/summarized
    as comments in the PR created for the proposal.
    * Iterate on the proposal based on the feedback received.
    * Once the proposal is approved by the maintainers, they can help merge the PR. The maintainers will also assign
    one or more contributors to implement and maintain the proposal.
* Implement and maintain the proposal.
    * The assigned contributors will implement the proposal in the KubeFleet project, and maintain it over time.
    * The implementation may span multiple release cycles, and may be broken down into multiple sub-tasks. Each sub-task
    should be tracked as an issue in the KubeFleet main repository, and linked to the original enhancement proposal issue.
    * If needed, contributors may submit follow-up PRs to edit the enhancement proposal to reflect any minor changes
    in the design or implementation of the proposal. 
    * The proposal issue can be closed once the implementation is completed.
