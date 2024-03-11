# Presto RFCs

Presto uses RFCs to propose major features.

## Proposing a Feature to Presto

To propose a new feature, you’ll submit a Request For Comments (RFC).  This RFC is basically a design proposal where
you can share a detailed description of what change you want to make, why it’s needed, and how you propose to implement
it.

It’s easier to make changes while your feature is in the ideation phase vs the PR phase, and this doc gives committers
an opportunity to suggest refinements before you start code.  For example, they may know of other planned efforts that
your work would otherwise collide with, or they may suggest implementation changes that make your feature more broadly
usable.

Smaller changes, including bug fixes and documentation improvements, can be implemented and reviewed via the normal
GitHub pull request workflow on the main Presto repo.

RFCs are more suitable for design proposals that are too large to discuss on a feature-request issue, like adding new
functionality to Presto, or if a discussion about the tradeoffs involved in a new addition are non-trivial.

If you are unsure whether something should be an RFC or a feature-request issue, you can ask by opening an issue in the
main [prestodb/presto](https://github.com/prestodb/presto) repository.

## How to add an RFC

See [CONTRIBUTING.md](CONTRIBUTING.md) for instructions on how to propose an RFC.