# The network is reliable™

When the topic of distributed systems and partition tolerance arises,
invariably one or more parties declares that in their extensive experience,
partitions never happen. Modern hardware and network architecture is so good
that communication failures are no longer the limiting risk in designing
distributed systems.

Those who have read (or learned through experience), some of Peter Deutsch's <a
href="http://www.rgoarchitects.com/files/fallacies.pdf">famous</a> <a
href="https://blogs.oracle.com/jag/resource/Fallacies.html">Fallacies of
Distributed Computing</a> will immediately disagree--but the burden of proof
positive falls upon them. We *assume* that our networks are reliable up until
they fail in a noticeable way. Few organizations rigorously *measure* network
latencies or the health of distributed systems, and those which do rarely
release their data.

We don't know much about networks because they're hard to measure. There are
*several* measures of latency in IP networks, from switching time to TCP RTT to
request latency. Latency doesn't tell us about the logical coherence of the
system: inconsistency, unavailability, or data loss. We can measure link
failures reliably, but whether those failures cause a mild increase in latency
or cause clients to lose data depends on topology, load, and timing. Still
harder to measure are partial failures like bursty network congestion or
temporarily stalled applications.

Because few applications can quantitatively evaluate (and alert their
operators) about inconsistency and data loss, it takes a catastrophic failure,
like widespread data corruption, a system refusing reads or writes, or extreme
latencies, in order for sysadmins to reliably notice a partition. Even then,
they may not identify the cause. And once someone has identified a failure,
they rarely publicize it. Nobody wants to blame their vendors publically, or
admit a loss of customer data if there's a way to handle it privately.

As a result, what we know about the failure modes in distributed systems is
part guesswork and part rumor. Sysadmins and developers will explain horrifying
failure modes to each other over a few beers, but detailed postmortems or
comprehensive surveys of availability are few and far between. In this post,
we'd like to bring these stories together, in an attempt to better understand
the risks of partitions in production systems.

## Hardware failure


