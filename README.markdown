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

## Low-level failures

## Power failure

<div class="accordion">
<h3>Fog Creek</h3>

As Microsoft's SIGCOMM paper suggests, redundancy doesn't always prevent link
failure. <a href="http://status.fogcreek.com/2011/06/postmortem.html">When a
power distribution unit failed</a> and took down one of two redundant
top-of-rack switches, Fog Creek lost service for a subset of customers on that
rack, but remained consistent and available for most users. However, the
*redundant* switch in that rack *also* lost power, for undetermined reasons.
That failure isolated the two neighboring racks from one another, <b>taking
down all On Demand services.</b>

</div>

## Network loops

<div class="accordion">
<h3>Fog Creek</h3>

<a href="http://status.fogcreek.com/2012/05/may-5-6-network-maintenance-post-mortem.html">During a planned network reconfiguration to improve reliability</a>, Fog Creek suddenly lost access to their network.

> A network loop had formed between several switches.
> 
> The gateways controlling access to the switch management network were
> isolated from each other, generating a split-brain scenario. Neither were
> accessible due to a sudden traffic flood. 
> 
> The flood was the result of a multi-switch BPDU (bridge protocol data unit)
> flood, indicating a spanning-tree flap. This is most likely what was changing
> the loop domain.

According to the BPDU standard, the flood *shouldn't have happened*. Unexpected
behavior outside the rules of the system caused <b>two hours of total
service unavailability.</b>

</div>

## Congestion and packet loss

<div class="accordion">
<h3>Freistil IT</h3>

Freistil IT hosts their servers with a colocation/managed-hosting provider.
Their monitoring system <a
href="http://www.freistil.it/2013/02/post-mortem-network-issues-last-week/">alerted
Freistil</a> to 50--100% packet loss, localized to a particular datacenter. The
network failure, caused by a router firmware bug, returned the next day.
Elevated packet loss caused the GlusterFS distributed filesystem to enter
split-brain undetected:

> Unfortunately, the malfunctioning network had caused additional problems
> which we became aware of in the afternoon when a customer called our support
> hotline because their website failed to deliver certain image files. We found
> that this was caused by a split-brain situation on the storage cluster
> “stor02″ where changes made on node “stor02b” weren’t reflected on “stor02a”
> and the self-heal algorithm built into the Gluster filesystem was not able to
> resolve this inconsistency between the two data sets."

Repairing that inconsistency led to a "brief overload of the web nodes because
of a short surge in network traffic".

</div>

## Internal partitions

<div class="accordion">
<h3>RelateIQ</h3>

In a comment on <a href="http://aphyr.com/posts/284-call-me-maybe-mongodb">Call
me maybe: MongoDB</a>, Scott Bessler observed exactly the same failure mode I
demonstrated in the Jepsen post:

> "Prescient. The w=safe scenario you show (including extra fails during
> rollback/re-election) happened to us today when EC2 West region had network
> issues that caused a network partition that separated PRIMARY from its 2
> SECONDARIES in a 3 node replset. 2 hours later the old primary rejoined and
> rolled back everything on the new primary. Our bad for not using w=majority."

This partition caused <b>two hours of write loss</b>. From my conversations
with large-scale MongoDB users, I gather that network events causing failover
on EC2 are common. While most folks report that running their own hardware is
more reliable, I know at least one company which sees their MongoDB cluster
fail over on a weekly basis.

</div>

## WAN failures

<div class="accordion">
<h3>PagerDuty</h3>

PagerDuty designed their system to remain available in the face of node, datacenter, or even *provider* failure; their services are replicated between two EC2 datacenters and another in Linode. <a href="http://blog.pagerduty.com/2013/04/outage-post-mortem-april-13-2013/">Degra TODO FINISH THIS BIT

</div>

## Global routing failure

<div class="accordion">
<h3>Cloudflare</h3>

CloudFlare runs 23 globally distributed datacenters with redundant network
paths and anycast failover. <a
href="http://blog.cloudflare.com/todays-outage-post-mortem-82515">In response
to a DDoS attack against one of their customers</a>, their operations team
deployed a new firewall rule to drop packets of a specific size. Juniper's
FlowSpec protocol propagated that rule to all CloudFlare edge routers, where:

> What should have happened is that no packet should have matched that rule
> because no packet was actually that large. What happened instead is that the
> routers encountered the rule and then proceeded to consume all their RAM
> until they crashed.

Recovering from the failure was more complicated by routers which failed to reboot automatically, and inaccessible management ports.

> Even though some data centers came back online initially, they fell back over
> again because all the traffic across our entire network hit them and
> overloaded their resources.

CloudFlare monitors their network carefully; the ops team had immediate
visibility of the failure. However, coordinating globally distributed systems
takes is complex, and calling on-site engineers to find and reboot routers by
hand takes time. Recovery began in 30 minutes and was complete after an hour of
unavailability.

</div>

## Misconfiguration

## CPU and GC pauses

<div class="accordion">
<h3>Bonsai.io</h3>

Not all partitions involve the network. Bonsai.io <a
href="http://www.bonsai.io/blog/2013/03/05/outage-post-mortem">discovered</a>
high CPU use and load averages on an ElasticSearch node. They restarted the
cluster, but it failed to converge, partitioning itself into two independent
components. The failure led to <b>20 minutes of hard downtime, and six hours of
degraded service.</b>

</div>

<div class="accordion">
<h3>Searchbox.io</h3>

Stop-the-world garbage collection can force application latencies on the order
of seconds to minutes. As Searchbox.io <a
href="http://blog.searchbox.io/blog/2013/03/03/january-postmortem">observed</a>,
GC pressure in an ElasticSearch cluster caused secondary nodes to declare a
primary dead and to attempt a new election. Because their configuration used an
improper value of `zen.minimum_master_nodes`, ElasticSearch was able to elect
two simultaneous primaries, leading to inconsistency and downtime. Configuring
distributed systems is difficult, and benign omissions can lead to serious
consequences.

</div>
