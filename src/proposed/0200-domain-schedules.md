<!--
  SPDX-License-Identifier: CC-BY-SA-4.0
  Copyright 2025 Proofcraft Pty Ltd

  Based on the Rust RFC template at <https://github.com/rust-lang/rfcs>
-->

# Runtime Domain Schedules

<!--
 - fill in proposal date
 - Fill in the rest of the sections. It is Ok to leave out sections that do not
   apply, but don't leave out sections lightly.

 - Make a pull request to <https://github.com/seL4/rfcs> to publish the RFC and
   start formal discussion.
-->

- Author: Gerwin Klein, Rafal Kolanski, Indan Zupancic
- Proposed: [YYYY-MM-DD fill in]

## Summary

We propose to add the ability to set, maintain, and switch between domain
schedules at runtime, authorised by the existing `DomainCap`, retaining the
current information flow proof and behaviour when the `DomainCap` no longer
exists in the system or is not used.

## Motivation

Currently the domain schedule is provided as a `.h` file and compiled into the
kernel. This was an initial stop-gap implementation and is inconvenient for SDK
approaches such as the Microkit and overall inflexible and hard to use.

For instance, it is currently not possible to provide a verified kernel binary
and modify it with a new kernel schedule without invalidating the verification.

It is also currently not possible to run a system in multiple modes, such as
"wheels up" and "wheels down" in aviation.

The intended use for the proposed mechanism is that the initialiser sets up
domain schedules at boot time, for instance the two schedules in the example
above, or a single domain schedule as would now be provided as `.h` file. After
that, if the system is to remain static, the `DomainCap` can be deleted as it is
now, or it can be given to a high privilege mode control component that can
atomically switch between schedules.

Updating a running system to tweak an existing schedule or add a new mode at
runtime would also become possible.

## Guide-level explanation

At a conceptual level, we propose to mostly keep the current domain scheduler
implementation as it is: a compile-time sized array of entries consisting of
duration and domain. Instead of representing a single schedule as before, we
propose to use the array now for representing multiple schedules. The new
components  are:

- entries with duration 0 are end markers for a domain schedule
- there is a new kernel global that contains the start index for the current
  domain schedule

### Example

The following diagram shows an example.

```none
-----------------------------------------------------------------
| (t_0, d_0) | (t_1, d_1) | ... | (0, 0) | (t_x, d_x) | .. | .. |
-----------------------------------------------------------------
       0          ^                  n         n+1

ksDomainStart = 0
ksDomScheduleIdx = 1
```

The example has an overall array of length `config_DomScheduleLength`,
configured at compile time, a current domain start index of 0, and an currently
active domain schedule index of 1. The current entry will run domain `d_1` for a
duration of `t_1 > 0`, and the schedule will keep going until it hits index `n`
which has an entry with duration 0. Instead of running the entry at index `n`,
it proceeds at `ksDomainStart`, i.e. index 0.

To populate the entires, a user thread with the `DomainCap` can invoke the
`setDomainEntry` method to set duration and domain for a specific index,
potentially with duration 0 to create an end marker.

To atomically switch to the second schedule in the example, a user thread with
the `DomainCap` can set `ksDomainStart` to `n+1`, which will start running
domain `d_x` for a duration `t_x`. The schedule will keep going until either
another entry with duration `0` occurs, or the index hits the end of the array.
In both cases, the index of the next entry will again be `ksDomainStart`.

### Conditions and Invariants

For both, setting the domain start and setting the value of an entry, the kernel
will prevent the user from creating a schedule where the entry at
`ksDomainStart` has duration 0. This would signify an empty domain schedule. It
would be possible to give this situation a useful meaning: not switching domains
until a new schedule is set. However, implementing it would be more invasive
than the currently proposed changes and the same effect can already be achieved
with a one-element schedule of long duration.

Since all duration 0 signifies an end marker, all active schedule entries
automatically have a duration > 0. On MCS, the duration at the API level is in
microseconds and must be >= MIN_PERIOD. The stored duration is in timer ticks.
On non-MCS configurations, the duration is measured in number of ticks (time
slices).

The kernel initialises with an array where all entries are `(0, 0)`, apart from
the entry at index 0, which will run domain 0 for the maximum expressible time.

## Reference-level explanation

### Invocations

The current invocations of the `DomainCap` remain as they are.
There are two new invocations:

#### setDomainStart

TODO

<!--
Explain switching behaviour (new domain starts running after this syscall)
-->

#### setDomainEntry

TODO

<!--
Explain effect of updating current entry (will be ignored until entry is read
next time in the schedule). Generally should avoid updating currently running
schedule, instead create new schedule only beyond current end marker.
-->

### Configuration Options

The new config option `config_DomScheduleLength` determines the static size of
the overall domain schedule array and thereby the longest domain schedule that
is possible to configure at runtime

<!--
Explain the change or feature as you would to the **developers and maintainers**
of the seL4 ecosystem. For instance, if it is a change to the seL4 API, this
section would contain the part that should go into API reference of the seL4
manual.

This section should provide sufficient technical detail to guide any related
implementation and ongoing maintenance. Where relevant, it should discuss
expected maintenace, performance, and verification impact.

This section should clearly describe how this change will interact with the
existing ecosystem, describe particular complex examples that may complicate the
implementation, and describe how the implementation should support the examples
in the previous section.
-->

## Drawbacks

TODO: breaking change, need to convert `.h` files into initialiser code. Could
adapt capDL with a schedule section, or provide a separate schedule
specification for initialiser components.

<!--
Outline any arguments that have been made against this proposal and discuss why
we may not want to accept it.  Also discuss any complications that may arise
from the proposed change that may require specific consideration.
-->

## Rationale and alternatives

TODO

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this?

## Prior art

TODO

Discuss prior art, both the good and the bad, in relation to this proposal.  A
few examples of what this can include are:

- For ecosystem proposals: Does this feature exist in similar systems and what
  experience have their community had?
- For community proposals: Is this done by some other community and what were
  their experiences with it?
- What lessons can we learn from what other communities have done here?
- Are there any published papers or great posts that discuss this? If you have
  some relevant papers to refer to, this can serve as a more detailed
  theoretical background.

This section is intended to encourage you as an author to think about the
lessons from other systems, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine -- your ideas are interesting to us
whether they are brand new or if it is an adaptation from other systems.

Note that while precedent set by other systems is some motivation, it does not
on its own motivate an RFC.

## Unresolved questions

TODO

- What needs to be resolved in further discussion before the RFC is approved?
- What needs to resolved during the implementation of this RFC?
- What related questions are beyond the scope of this RFC that should be
  addressed beyond its implementation?
