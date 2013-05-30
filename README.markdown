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

**PB This is a bit combative; what about the following:**

The topic of network partitions in distributed systems is contentious. Many will claim (often loudly) that real-world networks are reliable. In their experience, networks never partition, and modern hardware and network architecture render the risk of communication failures negligible. On the other hand, others subscribe to Peter Deutsch's <a href="https://blogs.oracle.com/jag/resource/Fallacies.html">Fallacies of
Distributed Computing</a> and will disagree. They'll attest that partitions have and do occur in their systems and that, as James Hamilton of Amazon AWS [neatly summarizes](http://perspectives.mvdirona.com/2010/04/07/StonebrakerOnCAPTheoremAndDatabases.aspx), “network partitions should be rare but net gear continues to cause more issues than it should.” Especially given the <a href="http://henryr.github.io/cap-faq/">consequences of partitions for distributed systems design</a>, who's right?

A key challenge in this debate is the lack of evidence supporting either side (admittedly, the burden of proof lies with the Deutsch camp). This is because partition behavior is extremely challenging to measure. There are *several* measures of latency in IP networks, from switching time to TCP RTT to request latency. **PB: well, can't effective partitions be detected by TCP timeouts?** inconsistency, unavailability, or data loss. We can measure individual link failures reliably, but whether those failures cause a mild increase in latency or cause hosts to lose connectivity on network topology, load, and timing. Still harder to measure are partial failures like bursty network congestion or temporarily stalled applications.

Because few applications can quantitatively evaluate (and alert their operators) about inconsistency and data loss, it often takes a catastrophic failure, such as prolonged service unavailability or widespread data corruption in order to notice a partition. **PB: again, not sure how hard to push here** Even then, owing to the complexity of modern services and networking infrastructure, operators may not be able identify a root cause. And even once someone has identified partition (or, in general, aberrant) behavior, they rarely publicize it. Nobody wants to blame their vendors publicly or admit a loss of customer data if there's a way to handle it privately.

**PB: I like our (this?) final paragraph a lot.**

As a result, much of what we know about the failure modes in real-wold distributed systems is founded on guesswork and rumor. Sysadmins and developers will explain horrifying failure modes to each other over a few beers, but detailed postmortems or comprehensive surveys of availability are few and far between. In this post, as an effort towards a more open and honest discussion of real-world partition behavior, we'd like to bring these stories together.

## Low-level failures

**PB Question: any reason to keep this separate from WAN failures? Can we reclassify it**

<div class="accordion">
<h3>Microsoft Datacenter Study</h3>

Researchers at Microsoft Research <a href="http://research.microsoft.com/en-us/um/people/navendu/papers/sigcomm11netwiser.pdf">studied the behavior</a> of network failures in several of their datacenters. They found an average failure rate of 5.2 devices per day and 40.8 links per day with a median time to repair of approximately five minutes (and up to one week). While the researchers note that correlating link failures and communication partitions is challenging, they estimate a median packet loss of 59,000 packets per failure. Perhaps more concerning is their finding that network redundancy improves median traffic by only 43%; that is, network redundancy does not eliminate many common causes of network failure.

</div>

<div class="accordion">
<h3>HP Enterprise Managed Networks</h3>

A joint study between researchers at University of California, San Diego and HP Labs <a href="http://www.hpl.hp.com/techreports/2012/HPL-2012-101.pdf">examined</a> the causes and severity of network failures in HP's managed networks by analyzing support ticket data. "Connectivity"-related tickets accounted for 11.4% of support tickets (14% of which were of the highest priority level), with a median incident duration of 2 hours and 45 minutes for the highest priority tickets and and a median duration of 4 hours 18 minutes for all priorities.

</div>

**PB note: we can move this**

<div class="accordion">
<h3>Google Chubby</h3>
Google's <a href="http://research.google.com/archive/chubby-osdi06.pdf">paper</a> describing the design and operation of Chubby, their distributed lock manager outlines the root causes of 61 outages over 700 days of operation across several clusters. Of the nine outages that lasted greater than 30 seconds, four were caused by network maintenance and two were caused by "suspected network connectivity problems."

</div>

<div class="accordion">
<h3>Amazon Dynamo</h3>
Amazon's <a href="http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf">Dynamo paper</a> frequently cites the incidence of partitions as a driving design consideration. Specifically, the authors note that they rejected designs from "traditional replicated relational database systems" because they "are not capable of handling network partitions."

</div>

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

**PB note: should we explain what we mean by "internal?" Should we pair this with the earlier stuff from Microsoft?**

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

<div class="accordion">
<h3>CENIC Study</h3>

Researchers at the University of California, San Diego <a
href="http://cseweb.ucsd.edu/~snoeren/papers/cenic-sigcomm10.pdf">quantitatively
analyzed</a> five years of operation CENIC wide-area network, which
contains over two hundred routers across California. By
cross-correlating link failures and additional external BGP and
traceroute data, they discovered over 508 "isolating network
partitions" that caused connectivity problems between hosts. Average
partition duration ranged from 6 minutes for software-related failures
to over 8.2 hours for hardware-related failures (median 2.7 and 32
minutes; 95th percentile of 19.9 minutes and 3.7 days).

</div>

<div class="accordion">
<h3>Yahoo! PNUTS/Sherpa</h3>
<a href="http://www.mpi-sws.org/~druschel/courses/ds/papers/cooper-pnuts.pdf">Yahoo! PNUTS/Sherpa</a> was designed as a distributed database operating out of multiple, geographically distinct sites. Originally, PNUTS supported a strongly consistent "timeline consistency" operation, with one master per data item. However, the developers <a href="http://developer.yahoo.com/blogs/ydn/sherpa-7992.html#4">noted that</a>, in the event of "network partitioning or server failures," this design decision was too restrictive for many applications:

 > The first deployment of Sherpa supported the timeline-consistency model — namely, all replicas of a record apply all updates in the same order — and has API-level features to enable applications to cope with asynchronous replication. Strict adherence leads to difficult situations under network partitioning or server failures. These can be partially addressed with override procedures and local data replication, but in many circumstances, applications need a relaxed approach."

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

<div class="accordion">
<h3>Global BGP Outages</h3>

There have been several global Internet outages related to BGP misconfiguration. Notably, in 2008, Pakistan Telecom, responding to a government edict to block YouTube.com, incorrectly advertised its (blocked) route to other provides, which hijacked traffic from the site and <a href="http://news.cnet.com/8301-10784_3-9878655-7.html">briefly rendered it unreachable</a>. In 2010, a group of Duke University researchers achieved similar effect by <a href="http://www.merit.edu/mail.archives/nanog/msg11505.html">testing</a> an experimental flag in the BGP protocol. Similar incidents have occured <a href="http://www.renesys.com/2006/01/coned-steals-the-net/">in 2006</a> (knocking sites like Martha Stewart Living and The New York Times offline), <a href="http://www.renesys.com/2005/12/internetwide-nearcatastrophela/">in 2005</a> (where a misconfiguration in Turkey attempted in a redirect for the *entire* internet), and <a href="http://merit.edu/mail.archives/nanog/1997-04/msg00380.html">in 1997</a>.

</div>

## Misconfiguration and Bugs

<div class="accordion">
<h3>Juniper Routing Bug</h3>
The software running inside network hardware (i.e., firmware) is subject to bugs just like the rest of computer software. A bug in a router upgrade in Juniper Networks's routers <a href="http://www.eweek.com/c/a/IT-Infrastructure/Bug-in-Juniper-Router-Firmware-Update-Causes-Massive-Internet-Outage-709180/">caused outages</a> in Level 3 Communications's networking backbone. This subsequently knocked services like Time Warner Cable and RIM BlackBerry, and several UK internet service providers offline.

</div>

## CPU and GC pauses

<div class="accordion">
<h3>Bonsai.io</h3>

Not all partitions involve the network hardware directly but can surface due to software inability to process messages. Bonsai.io <a
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
