# Architecture

The purpose of this repo is to hold architecture documents that relate to overall architecture and design of Kuadrant.
Note currently this repo is being updated and for know it is better to look at the individual components in this repo

## What belongs here:

- Cross cutting proposal that involves or impacts on more than one component within Kuadrant
- Documents describing the overall architecture or a sub context (control plane / data plane) within the architecture 

## Kuadrant RFCs

The RFC (Request For Comments) process is aiming at providing a consistent and well understood way of adding new features or introducing breaking changes in the Kuadrant stack. It provides a mean for all stakeholders and community at large to provide feedback and be confident about the evolution of our solution.

Many, if not most of the changes will not require to follow this process. Bug fixes, refactoring, performance improvements or documentation additions/improvements can be implemented using the tradition PR (Pull Request) model straight to the targeted repositories on Github. 

Additions or any other changes that impact the end user experience will need to follow this process.

### When is an RFC required?

This process is meant for any changes that affect the user's experience in any way: addition of new APIs, changes to existing APIs - whether they are backwards compatible or not - and any other change to behaviour that affects the user of any components of Kuadrant.

 - API additions;
 - API changes;
 - … any change in behaviour.

#### When _no_ RFC is required?

- bugfixes;
- refactoring; 
- performance improvements.


### The RFC process

The first step in adding a new feature to Kuadrant, or a starting a major change, is the having a RFC merged into the [repository](https://github.com/Kuadrant/architecture). One the file has been merged, the RFC is considered _active_ and ready to be worked on.

1. Fork the RFC repo
1. Use the template `0000-template.md` to copy and rename it into the `rfcs` directory. Change the `template` suffix to something descriptive. But this is still a proposal and as no assigned RFC number to it yet.
1. Fill the template out. Try to be as thorough as possible. While some sections may not apply to your feature/change request, try to complete as much as possible, as this will be the basis for further discussions. Consult the [definitions](definitions.md) for useful terminology guidelines.
1. Submit a pull request for the proposal. That's when the RFC is open for actual comments by other members of team and the broader community.
1. The PR is to be handled just like a "code PR", wait on people's review and integrate the feedback provided. These RFCs can also be discussed during our weekly technical call meeting, yet the summary would need to be captured on the PR.
1. How ever much the orignal proposal changes during this process, never force push or otherwise squash the history, or even rebase your branch. Try keeping the commit history as clean as possible on that PR, in order to keep a trace of how the RFC evolved.
1. Once all point of views have been shared and input has been integrated in the PR, the author can push the RFC in the _final comment period_ (FCP) which lasts a week. This is the last chance for anyone to provide input. If during the FCP, consensus cannot be reached, it can be decided to extend that period by another week. Consensus is achieved by getting two approvals from the core team.
1. As the PR is merged, it gets a number assigned, making the RFC _active_. 
1. If on the other hand the consensus is to _not_ implement the feature as discussed, the PR is closed.

### The RFC lifecycle

- _Open_: A new RFC as been submitted as a proposal
- _FCP_: Final comment period of one week for last comments
- _Active_: RFC got a number assigned and is ready for implementation with the work tracked in an issue, which summarizes the state of the implementation work.

#### Implementation

The work is itself tracked in a "master" issue with all the individual, manageable implementation tasks tracked. 
The state of that issue is initially "open" and ready for work, which doesn't mean it'd be worked on immediately or by the RFC's author. That work will be planned and integrated as part of the usual release cycle of the Kuadrant stack. 

#### Amendments

It isn't expected for an RFC to change, once it has become _active_. Minor changes are acceptable, but any major change to an active RFC should be treated as an independent RFC and go through the cycle described here.


## What doesn't belong here:

- Specific component level documents that relate to the internal design of a component, these should reside within individual component repos
- Development process docs for individual components


## Technical Discussion and Community meetings

- [Agenda](./meetings/agenda.md)
