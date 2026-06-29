# KubeFleet Enhancement Proposals

This repository tracks enhancement proposals for the KubeFleet project, a cloud-native solution for multi-cluster
management, as inspired by the [Kubernetes enhancement process](https://github.com/kubernetes/enhancements).

The repository helps facilitate and track discussions and decisions on how KubeFleet shall evolve over time,
in the form of actionable design documents. The enhancements, once approved, become part of the KubeFleet roadmap,
and are implemented in the project by stakeholders, possibly across multiple release cycles.

Anyone is welcome to create an enhancement proposal; to get started, see our [Contributing Guidelines](CONTRIBUTING.md)
for more information. You can find the list of approved enhancement proposals in the [enhancements](enhancements) directory;
look up active proposals under discussion in the list of issues and pull requests of this repository, and join
the conversation to help make the proposal better.

We are still iterating on our enhancement proposal process. If you have any concern, feedback, or suggestion, please
submit an issue in this repository or reach out to us via [our community channels](https://github.com/kubefleet-dev/community).

## Why track enhancement proposals?

As KubeFleet continues to evolve, it is important for the community to have a clear, shared understanding on
how we design, build, test, and document new features and functionalities. We track enhancement proposals in this
repository to help ensure that the community can collaborate effectively on the evolution of KubeFleet, depend on
each other to find the best solutions forward, and build consensus on the direction of the project, before
investing any significant engineering effort.

## Is my thing an enhancement?

As a rule of thumb, consider creating a KubeFleet enhancement proposal if you have an idea about KubeFleet that:

* introduces API changes in the project
* adds new features or functionality to the project that can become something users can depend on
* involves substantial changes to the architecture and/or implementation of the project
* impacts the user experience or system behavior of the project in a significant way
* has upgrade/downgrade implications for existing installations

It is probably OK to not write an enhancement proposal if you just plan to:

* fix a bug
* add more tests
* refactor existing code without changing its behavior
* document best practices, useful patterns/cases, or instructions to use the project
* implement a minor change that has minimal impact and/or has no user-visible effect
* make performance improvements that are only visible to users as faster operations

If you are not sure about whether your idea requires an enhancement proposal, feel free to file an issue in the repository
and ask the community for feedback.

On the other hand, if you have an idea that needs an enhancement proposal, but you are not yet ready to have one
written or you do not have the complete design in mind for now, you can also submit a feature request instead in the
[KubeFleet main repository](https://github.com/kubefleet-dev/kubefleet/issues). 

## When to create a new enhancement proposal

It is recommended that one create a new enhancement proposal when they have:

* shared and discussed the idea with the community to gather feedback and support;
* (optionally) created a prototype that validates the concept (if applicable);
* identified stakeholders who would be interested in implementing and maintaining the enhancement;
* recognized that it may take significant engineering effort that spans multiple release cycles to complete the enhancement;

If you need help, ask via [our community channels](https://github.com/kubefleet-dev/community).
