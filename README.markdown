<script type="text/javascript">
// Remember this script tag
var scripts = document.getElementsByTagName('script');
var script = scripts[scripts.length - 1];
var header = 'h2';

// When loaded, post-process document:
$(document).ready(function() {
  var article = $(script).parent();
  var sections = article.children(header);
  sections.slice(0, -1).each(function() {
    $(this).css({cursor: 'pointer'});
    $(this).prepend("➤ ");
    $(this).nextUntil(header).wrapAll('<div class="more" style="display: none" />');
    $(this).click(function() {
      $(this).next('.more').slideToggle();
    });
  });
});


</script>

*I've been discussing <a href="/tags/jepsen">Jepsen</a> and partition
 tolerance with <a href="http://www.bailis.org/">Peter Bailis</a>,
 who I invited to join me in writing this post.*


Network partitions are a contentious subject. Some claim that modern
networks are reliable and that we are too concerned with designing for
*theoretical* failure modes. They often accept that single-node
failures are common, but argue that we can <a
href="http://blog.voltdb.com/clarifications-cap-theorem-and-data-related-errors/">reliably
detect and handle thems</a>. Conversely,
others <a
href="http://www.rgoarchitects.com/files/fallacies.pdf">subscribe</a>
to Peter Deutsch's <a
href="https://blogs.oracle.com/jag/resource/Fallacies.html">Fallacies
of Distributed Computing</a> and disagree. They'll attest that
partitions do occur in their systems, and that, as James Hamilton of
Amazon Web Services [neatly
summarizes](http://perspectives.mvdirona.com/2010/04/07/StonebrakerOnCAPTheoremAndDatabases.aspx),
"network partitions should be rare but net gear continues to cause
more issues than it should." The answer to this debate <a
href="http://henryr.github.io/cap-faq/">radically affects</a> the design of distributed databases, queues, and applications. So who's right?

A key challenge in this dispute is the lack of evidence. We have few normalized bases for comparing network and application reliability and even less data for each. We can track link availability and estimate packet loss, but understanding the end-to-end effect on *applications* is more difficult. The scant evidence we have is difficult to generalize: it is often deployment-specific and closely tied to particular vendors, topologies, and application designs. Worse, even when an organization has clear picture of their network's behavior, they rarely share specifics. Finally, distributed systems are designed to resist failure, which means *noticeable* outages often depend on complex interactions of failure modes. Many applications silently degrade when the network fails, and resulting problems may not be understood for some time, when they are noticed at all.

As a result, much of what we know about the failure modes in real-wold
distributed systems is founded on guesswork and rumor. Sysadmins and developers
will quietly discuss failure modes over beers, but detailed, public postmortems and comprehensive surveys of network availability are few and far between. In this post, we'd like to bring a few of these stories together. We believe this is a first step towards a more open and honest discussion of real-world partition behavior, and, ultimately, more robust distributed systems designs.

## Hints from big deployments

To start off, let's consider evidence from big players in
distributed systems: companies running globally distributed infrastructure with
hundreds of thousands of nodes. Of all of the data we have collected, these reports best summarize operation in the large, distilling the experience of operating what are likely the largest distributed systems ever deployed. Their publications, which (unlike many of the case studies we will examine later) often capture aggregate system behavior and large-scale statistical trends, indicate (often obliquely) that partitions are a reality in their deployments.

### The Microsoft datacenter Study

Microsoft Research <a
href="http://research.microsoft.com/en-us/um/people/navendu/papers/sigcomm11netwiser.pdf">studied
the behavior</a> of network failures in several of their datacenters. They
found an average failure rate of 5.2 devices per day and 40.8 links per day
with a median time to repair of approximately five minutes (and up to one
week). While the researchers note that correlating link failures and
communication partitions is challenging, they estimate a median packet loss of
59,000 packets per failure. Perhaps more concerning is their finding that
network redundancy improves median traffic by only 43%; that is, network
redundancy does not eliminate many common causes of network failure.


### HP Enterprise Managed Networks

A joint study between researchers at University of California, San Diego and HP
Labs <a
href="http://www.hpl.hp.com/techreports/2012/HPL-2012-101.pdf">examined</a> the
causes and severity of network failures in HP's managed networks by analyzing
support ticket data. "Connectivity"-related tickets accounted for 11.4% of
support tickets (14% of which were of the highest priority level), with a
median incident duration of 2 hours and 45 minutes for the highest priority
tickets and and a median duration of 4 hours 18 minutes for all priorities.


### Google Chubby

Google's <a
href="http://research.google.com/archive/chubby-osdi06.pdf">paper</a>
describing the design and operation of Chubby, their distributed lock manager
outlines the root causes of 61 outages over 700 days of operation across
several clusters. Of the nine outages that lasted greater than 30 seconds, four
were caused by network maintenance and two were caused by "suspected network
connectivity problems."


### Google's Design Lessons from Distributed Systems

In <a
href="http://www.cs.cornell.edu/projects/ladis2009/talks/dean-keynote-ladis2009.pdf">Design
Lessons and Advice from Building Large Scale Distributed Systems</a>, Jeff Dean
suggests that a typical first year for a new Google cluster involves:

- 5 racks going wonky (40-80 machines seeing 50% packet loss)
- 8 network maintenances (4 might cause ~30-minute random connectivity losses)
- 3 router failures (have to immediately pull traffic for an hour)

While Google doesn't tell us much about the application-level consequences of
their network partitions, "Lessons From Distributed Systems" suggests it they are a significant concern, citing the challenge of "[e]asy-to-use abstractions for
resolving conflicting updates to multiple versions of a piece of state" as
being useful for "reconciling replicated state in different data centers after
repairing a network partition."


### Amazon Dynamo

Amazon's <a
href="http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf">Dynamo
paper</a> frequently cites the incidence of partitions as a driving design
consideration. Specifically, the authors note that they rejected designs from
"traditional replicated relational database systems" because they "are not
capable of handling network partitions."


### Yahoo! PNUTS/Sherpa
<a
href="http://www.mpi-sws.org/~druschel/courses/ds/papers/cooper-pnuts.pdf">Yahoo!
PNUTS/Sherpa</a> was designed as a distributed database operating out of
multiple, geographically distinct sites. Originally, PNUTS supported a strongly
consistent "timeline consistency" operation, with one master per data item.
However, the developers <a
href="http://developer.yahoo.com/blogs/ydn/sherpa-7992.html#4">noted that</a>,
in the event of "network partitioning or server failures," this design decision
was too restrictive for many applications:

> The first deployment of Sherpa supported the timeline-consistency model —
> namely, all replicas of a record apply all updates in the same order — and
> has API-level features to enable applications to cope with asynchronous
> replication. Strict adherence leads to difficult situations under network
> partitioning or server failures. These can be partially addressed with
> override procedures and local data replication, but in many circumstances,
> applications need a relaxed approach."



## GC, Disk IO, and CPU load

Not all partitions originate in the physical network. Sometimes dropped or delayed
messages are a consequence of crashes, race conditions, OS scheduler latency,
or overloaded processes. The following studies highlight the fact that partitions--wherein the system delays or drops messages--can occur at any layer of the software stack.

### High CPU use

Bonsai.io <a
href="http://www.bonsai.io/blog/2013/03/05/outage-post-mortem">discovered</a>
high CPU use and load averages on an ElasticSearch node. They restarted the
cluster, but upon restart, the cluster partitioned itself into two independent
components. The failure led to 20 minutes of unavailability, and six hours of
degraded service.


### Long GC pauses

Stop-the-world garbage collection can force application latencies on the order
of seconds to minutes. As Searchbox.io <a
href="http://blog.searchbox.io/blog/2013/03/03/january-postmortem">observed</a>,
GC pressure in an ElasticSearch cluster can cause secondary nodes to declare a
primary dead and to attempt a new election. Because their configuration used a low value of `zen.minimum_master_nodes`, ElasticSearch was able to elect
two simultaneous primaries, leading to inconsistency and downtime.


### MySQL overload and a Pacemaker segfault

Github relies heavily on Pacemaker and Heartbeat: programs which coordinate
cluster resources between nodes. They use Percona Replication Manager, a
resource agent for Pacemaker, to replicate their MySQL database between three
nodes.

On September 10th, 2012, a routine database migration caused unexpectedly high
load on the MySQL primary. Percona Replication Manager, unable to perform
health checks against the busy MySQL instance, decided the primary was down and
promoted a secondary. The secondary had a cold cache and performed poorly.
Normal query load on the node caused it to slow down, and Percona failed *back*
to the original primary. The operations team put Pacemaker into
maintenance-mode, temporarily halting automatic failover. The site appeared to
recover.

The next morning, the operations team discovered that the standby MySQL node
was no longer replicating changes from the primary. Operations decided to
disable Pacemaker's maintenance mode to allow the replication manager to fix
the problem.

> Upon attempting to disable maintenance-mode, a Pacemaker segfault occurred
> that resulted in a cluster state partition. After this update, two nodes
> (I'll call them 'a' and 'b') rejected most messages from the third node
> ('c'), while the third node rejected most messages from the other two.
> Despite having configured the cluster to require a majority of machines to
> agree on the state of the cluster before taking action, two simultaneous
> master election decisions were attempted without proper coordination. In the
> first cluster, master election was interrupted by messages from the second
> cluster and MySQL was stopped.

> In the second, single-node cluster, node 'c' was elected at 8:19 AM, and any
> subsequent messages from the other two-node cluster were discarded. As luck
> would have it, the 'c' node was the node that our operations team previously
> determined to be out of date. We detected this fact and powered off this
> out-of-date node at 8:26 AM to end the partition and prevent further data
> drift, taking down all production database access and thus all access to
> github.com.

The partition caused inconsistency in the MySQL database--both internally and
between MySQL and other data stores, like Redis. Because foreign key
relationships were not consistent, Github showed private repositories to the
wrong user's dashboards, and incorrectly routed some newly created repos.

Github thought carefully about their infrastructure design, and were still
surprised by a complex interaction of partial failures and software bugs. As
they note in the postmortem:

> ... if any member of our operations team had been asked if the failover
> should have been performed, the answer would have been a resounding
> <b>no</b>.

Distributed systems are *hard*.



## NICs and drivers

### BCM5709 and friends

Unreliable NIC hardware or drivers are implicated in a broad array of
partitions. <a href="http://www.spinics.net/lists/netdev/msg210485.html">Marc
Donges and Michael Chan</a> bring us a thrilling report of the popular BCM5709
chipset abruptly dropping inbound *but not outbound* packets to a machine.
Because the NIC dropped inbound packets, the node was unable to service requests.
However, because it could still *send* heartbeats to its hot spare via keepalived, the
spare considered the primary alive and refused to take over. The service was unavailable
for five hours and did not recover without a reboot.

Sven Ulland <a
href="http://www.spinics.net/lists/netdev/msg210491.html">followed up</a>,
reporting the same symptoms with the BCM5709S chipset on Linux
2.6.32-41squeeze2. Despite pulling commits from mainline which supposedly fixed
a similar set of issues with the bnx2 driver, they were unable to resolve the
issue until version 2.6.38.

Since Dell shipped a large number of servers with the Broadcom BCM5709, the
impact of these firmware bugs was widely observed. For instance, the 5709 had a bug
in its <a
href="http://monolight.cc/2011/08/flow-control-flaw-in-broadcom-bcm5709-nics-and-bcm56xxx-switches/">802.3x
flow control code</a> causing them to spew PAUSE frames when the
chipset crashed or its buffer filled out. This problem was magnified by the
BCM56314 and BCM56820 switch-on-a-chip devices (a component in a number of
Dell's top-of-rack switches), which, by default, spewed PAUSE frames at *every*
interface trying to communicate with the offending 5709 NIC. This led to
cascading failures on entire switches or networks.

The bnx2 driver could also cause transient or flapping network failures, as
described in this <a
href="http://elasticsearch-users.115913.n3.nabble.com/Cluster-Split-Brain-td3333510.html">ElasticSearch
split brain report</a>. Meanwhile, the the Broadcom 57711 was notorious for
causing <a
href="http://communities.vmware.com/thread/284628?start=0&tstart=0">extremely
high latencies under load with jumbo frames</a>, a particularly thorny issue
for ESX users with iSCSI-backed storage.


### A GlusterFS partition caused by a driver bug

After a scheduled upgrade, <a
href="https://www.citycloud.eu/cloud-computing/post-mortem/">CityCloud noticed
unexpected network failures</a> in two distinct GlusterFS pairs</a>, followed
by a third. Suspecting link aggregation, CityCloud disabled the feature on
their switches and allowed self-heal operations to proceed.

Roughly 12 hours later, the network failures returned on one node. CityCloud
identified the cause as driver issue and updated the downed node, returning
service. However, the outage resulted in data inconsistency between GlusterFS
pairs:

> As the servers lost storage abruptly there were certain types of Gluster
> issues where files did not match each other on the two nodes in each storage
> pair. There were also some cases of data corruption in the VMs filesystems
> due to VMs going down in an uncontrolled way.


## Datacenter network failures

Individual interfaces can fail, but those typically appear as single-node
outages. A more dangerous problem are failures somewhere in the physical
network. Switches are subject to power failure, misconfiguration, firmware
bugs, topology changes, cable failures, and malicious traffic. Their failure
modes are accordingly diverse:

### Power failure on both redundant switches

As Microsoft's SIGCOMM paper suggests, redundancy doesn't always prevent link
failure. <a href="http://status.fogcreek.com/2011/06/postmortem.html">When a
power distribution unit failed</a> and took down one of two redundant
top-of-rack switches, Fog Creek lost service for a subset of customers on that
rack but remained consistent and available for most users. However, the
other switch in that rack *also* lost power for undetermined reasons.
That failure isolated the two neighboring racks from one another, taking
down all On Demand services.


### Switch split-brain caused by BPDU flood

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

According to the BPDU standard, the flood *shouldn't have happened*. But it did, and this unexpected behavior that deviated from the system assumptions resulted in two hours of total service
unavailability.


### Bridge loops, misconfiguration, broken MAC caches

In an effort to address high latencies caused by a daisy-chained network
topology, Github <a
href="https://github.com/blog/1346-network-problems-last-friday">installed a
set of aggregation switches</a> in their datacenter. Despite a redundant
network, the installation process resulted in bridge loops, and switches
disabled links to prevent failure. This problem was quickly resolved, but later
investigation revealed that many interfaces were still pegged at 100% capacity.

While investigating that problem, a misconfigured switch triggered aberrant automatic fault detection behavior: when one link was disabled, the fault detector disabled *all* links. This caused 18 minutes of hard downtime. The problem was later traced to a firmware bug preventing switches from updating their MAC address caches correctly, which forced them to broadcast most packets to every interface. 


### Mystery RabbitMQ partitions

Sometimes, nobody knows why the system partitioned. This <a
href="http://serverfault.com/questions/497308/rabbitmq-network-partition-error">RabbitMQ
failure</a> seems like one of those cases: few retransmits, no large gaps
between messages, and no clear loss of connectivity between nodes. Upping the
partition detection timeout to 2 minutes reduced the frequency of partitions
but didn't prevent them altogether. 


### DRBD split-brain

When a two-node cluster partitions, there are no cases in which a node can
reliably declare itself to be the primary. When this happens to a DRBD filesystem, <a
href="http://serverfault.com/questions/485545/dual-primary-ocfs2-drbd-encountered-split-brain-is-recovery-always-going-to-be">as one user reported</a>, both nodes can remain online and accept writes, leading
to divergent filesystem-level changes. The only realistic option for resolving
these kinds of conflicts is to discard all writes not made to a selected
component of the cluster.


### A NetWare split-brain

Intermittent failures can lead to long outages. In this <a
href="http://novell.support.cluster-services.free-usenet.eu/Split-Brain-Condition_T31677168_S1">Usenet
post to novell.support.cluster-services</a>, an admin reports their two-node
failover cluster running Novell NetWare experienced transient network outages.
The secondary node eventually killed itself, and the primary (though still
running) was no longer reachable by other hosts on the network. The post goes
on to detail a series of network partition events correlated with backup jobs!


### MLAG, Spanning Tree, and STONITH

Github writes great postmortems, and this one is no exception. On <a
href="https://github.com/blog/1364-downtime-last-saturday">December 22nd,
2012</a>, a planned software update on an aggregation switch caused some mild
instability during the maintenance window. In order to collect diagnostic
information about the instability, the network vendor killed a particular
software agent running on one of the aggregation switches.

Github's aggregation switches are clustered in pairs using a feature called
MLAG, which presents two physical switches as a single layer 2 device. The MLAG
failure detection protocol relies on *both* ethernet link state *and* a logical
heartbeat message exchanged between nodes. When the switch agent was killed, it
was *unable* to shut down the ethernet link. Unlucky timing confused the MLAG
takeover, preventing the still-healthy agg switch from handling link
aggregation, spanning-tree, and other L2 protocols as normal. This forced a
spanning-tree leader election and reconvergence for all links, *blocking all
traffic between access switches for 90 seconds*.

The 90-second network partition caused fileservers using Pacemaker and DRBD for
HA failover to declare each other dead, and to issue STONITH (Shoot The Other
Node In The Head) messages to one another. The network partition delayed
delivery of those messages, causing some fileserver pairs to believe they were
*both* active. When the network recovered, both nodes shot each other at the
same time. With both nodes dead, files belonging to that pair were unavailable.

To prevent filesystem corruption, DRBD requires that administrators ensure the
original primary node is still the primary node before resuming replication.
For pairs where both nodes were primary, the ops team had to examine log files
and/or bring the node online in isolation to determine its state. Recovering those downed fileserver pairs took five hours, during which Github service was significantly degraded.


## Managed hosting providers

Running your own datacenter can be cheaper and more reliable than using public cloud infrastructure, but it also means you have to be a network and server administrator. What about managed hosting providers, which rent dedicated or virtualized hardware to users and often take care of the network and hardware setup for you?

### An undetected GlusterFS split-brain

Freistil IT hosts their servers with a colocation/managed-hosting provider.
Their monitoring system <a
href="http://www.freistil.it/2013/02/post-mortem-network-issues-last-week/">alerted
Freistil</a> to 50--100% packet loss localized to a specific datacenter. The
network failure, caused by a router firmware bug, returned the next day.
Elevated packet loss caused the GlusterFS distributed filesystem to enter
split-brain undetected:

> Unfortunately, the malfunctioning network had caused additional problems
> which we became aware of in the afternoon when a customer called our support
> hotline because their website failed to deliver certain image files. We found
> that this was caused by a split-brain situation on the storage cluster
> “stor02″ where changes made on node “stor02b” weren’t reflected on “stor02a”
> and the self-heal algorithm built into the Gluster filesystem was not able to
> resolve this inconsistency between the two data sets.

Repairing that inconsistency led to a "brief overload of the web nodes because
of a short surge in network traffic."


### An anonymous hosting provider

From what we can gather informally, *all* the major managed hosting providers
experience regular network failures. One company running 100-200 nodes on a
major hosting provider reported that in a 90-day period the provider's network
went through five distinct periods of partitions. Some partitions disabled connectivity between the provider's cloud network and the public internet, and others separated the cloud network from the provider's internal managed-hosting network. The failures caused unavailability but, because this company wasn't running any significant distributed systems *across* those partitioned networks, there were no major inconsistencies.


### Pacemaker/Heartbeat split-brain

A post to Linux-HA <a
href="http://readlist.com/lists/lists.linux-ha.org/linux-ha/6/31964.html">details a long-running partition between a Heartbeat pair</a>,
in which two Linode VMs have each declared the other dead and claimed a
shared IP for themselves. Successive posts suggest further network problems:
emails failed to dispatch due to DNS resolution failure, and nodes reported
"network unreachable." In this case, the impact appears to have been minimal--in
part because the partitioned application was just a proxy.



## Cloud providers

Large-scale virtualized environments are notorious for transient latency,
dropped packets, and full-blown network partitions, often affecting a particular software version or availability zone. Sometimes the failures occur between specific subsections of the provider's datacenter, revealing internal planes of cleavage.

### An isolated MongoDB primary on EC2

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
on EC2 are common. Simultaneous primaries accepting writes for *multiple days*
are not unknown. 

### Mnesia split-brain on EC2

EC2 outages can leave two nodes connected to the internet but unable to see
each other. This type of partition is especially dangerous, as writes to both
sides of a partitioned cluster can cause inconsistency and lost data. That's
exactly what happened to <a
href="http://dukesoferl.blogspot.com/2008/03/network-partition-oops.html?m=1">this
Mnesia cluster</a>, which diverged overnight. Their state wasn't critical, so
the operations team simply nuked one side of the cluster. They conclude: "the
experience has convinced us that we need to prioritize up our network partition
recovery strategy".


### EC2 instability causing MongoDB and ElasticSearch unavailability

Network disruptions in EC2 can affect only certain groups of nodes.
For instance, <a
href="https://forums.aws.amazon.com/thread.jspa?messageID=454155">this report
of a total partition between the frontend and backend stacks</a> states that
their the web servers lose their connections to all backend instances for a few
seconds, several times a month. Even though the disruptions were short, cluster
convergence resulted in 30-45 minute outages and a corrupted index for
ElasticSearch. As problems escalated, the outages occurred "2 to 4 times a
day."


### VoltDB split-brain on EC2

One VoltDB user reports <a
href="https://forum.voltdb.com/showthread.php?552-Nodes-stop-talking-to-each-other-and-form-independent-clusters">regular
network failures causing replica divergence</a> but also reported that
their network logs included no dropped packets. Because this cluster had not
enabled split-brain detection, both nodes ran as isolated primaries,
causing significant data loss. 


### ElasticSearch discovery failure on EC2

<a
href="http://elasticsearch-users.115913.n3.nabble.com/EC2-discovery-leads-to-two-masters-td3239318.html">Another
EC2 split-brain</a>: a two-node cluster failed to converge on "roughly 1 out of
10 startups" when discovery messages took longer than three seconds to
exchange. As a result, both nodes would start as masters with the same cluster
name. Since ElasticSearch doesn't demote primaries automatically, split-brain
persisted until administrators intervened. Upping the discovery timeout to 15
seconds resolved the issue.


### RabbitMQ and ElasticSearch on Windows Azure

There are a few <a
href="http://social.msdn.microsoft.com/Forums/en-US/WAVirtualMachinesforWindows/thread/b261e1aa-5ec4-42fc-80ef-5b50a0a00618">scattered
reports of Windows Azure partitions, such as <a
href="http://rabbitmq.1065348.n5.nabble.com/Instable-HA-cluster-td24690.html">this
account</a> of a RabbitMQ cluster which entered split-brain on a weekly basis.
There's also an account of <a
href="https://groups.google.com/forum/?fromgroups#!topic/elasticsearch/muZtKij3nUw">ElasticSearch
split-brain</a>, but since Azure is a relative newcomer compared to EC2, descriptions of its network reliability are limited.


### AWS EBS outage

On April 21st, 2011, <a href="http://aws.amazon.com/message/65648/">Amazon's
Web Services</a> went down for over 12 hours, causing outages for hundreds of
high-profile web sites to go offline. As a part of normal AWS scaling
activities, Amazon engineers shifted traffic away from a router in the Elastic
Block Store (EBS) network in a single US-East Availability Zone (AZ).

> The traffic shift was executed incorrectly and rather than routing the
> traffic to the other router on the primary network, the traffic was routed
> onto the lower capacity redundant EBS network. For a portion of the EBS
> cluster in the affected Availability Zone, this meant that they did not have
> a functioning primary or secondary network because traffic was purposely
> shifted away from the primary network and the secondary network couldn’t
> handle the traffic level it was receiving. As a result, many EBS nodes in the
> affected Availability Zone were completely isolated from other EBS nodes in
> its cluster.  Unlike a normal network interruption, this change disconnected
> both the primary and secondary network simultaneously, leaving the affected
> nodes completely isolated from one another.

The partition coupled with aggressive failure-recovery code caused a mirroring
storm, which led to network congestion and triggered a previously unknown race
condition in EBS. EC2 was unavailable for roughly 12 hours, and EBS was unavailable or degraded for over 80 hours.

The EBS failure also caused an outage in Amazon's Relational Database Service.
When one AZ fails, RDS is designed to fail over to a different AZ. However,
2.5% of multi-AZ databases in US-East failed to fail over due to "stuck" IO.

> The primary cause was that the rapid succession of network interruption
> (which partitioned the primary from the secondary) and “stuck” I/O on the
> primary replica triggered a previously un-encountered bug.  This bug left the
> primary replica in an isolated state where it was not safe for our monitoring
> agent to automatically fail over to the secondary replica without risking
> data loss, and manual intervention was required."

This correlated failure caused widespread outages for clients relying
on AWS). For example, <a href="https://status.heroku.com/incidents/151">Heroku
reported</a> between 16 and 60 hours of unavailability for their users'
databases.


## WAN failures

While we have largely focused on failures over local area networks (or near-local networks), wide area network (WAN) failures are also common (if less frequently documented). These failures are particularly interesting because there are often fewer redundant WAN links and systems guaranteeing high availability (and catastrophic disaster recovery) require distribution across multiple availability zones or datacenters. Accordingly, graceful degradation of system behavior under partitions (not to mention increased latency) is especially important for many systems operating in the wide area.

### PagerDuty

PagerDuty designed their system to remain available in the face of node,
datacenter, or even *provider* failure; their services are replicated between
two EC2 regions and a datacenter hosted by Linode. On April 13, 2013, <a
href="http://blog.pagerduty.com/2013/04/outage-post-mortem-april-13-2013/">an
AWS peering point in northern California degraded</a>, causing connectivity
issues for one of PagerDuty's EC2 nodes. As latencies between AWS availability
zones rose, the notification dispatch system lost the ability to contact to a quorum of nodes and stopped dispatching messages entirely.

Even though PagerDuty's infrastructure was designed with partition tolerance in
mind, correlated failures in two Amazon AZs caused 18 minutes of
unavailability, dropping inbound API requests and delaying queued pages until
quorum was re-established.


### CENIC Study

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



## Global routing failure

Some network failures take place on a globally distributed scale.

### Cloudflare

CloudFlare runs 23 datacenters with redundant network paths and anycast
failover. <a
href="http://blog.cloudflare.com/todays-outage-post-mortem-82515">In response
to a DDoS attack against one of their customers</a>, their operations team
deployed a new firewall rule to drop packets of a specific size. Juniper's
FlowSpec protocol propagated that rule to all CloudFlare edge routers, but then--

> What should have happened is that no packet should have matched that rule
> because no packet was actually that large. What happened instead is that the
> routers encountered the rule and then proceeded to consume all their RAM
> until they crashed.

Recovering from the failure was more complicated by routers which failed to
reboot automatically, and inaccessible management ports.

> Even though some data centers came back online initially, they fell back over
> again because all the traffic across our entire network hit them and
> overloaded their resources.

CloudFlare monitors their network carefully; the ops team had immediate
visibility into the failure. However, coordinating globally distributed systems
takes is complex, and calling on-site engineers to find and reboot routers by
hand takes time. Recovery took 30 minutes to begin, and was complete after an
hour of unavailability.

### Juniper routing bug

A firmware bug introduced as a part of an upgrade in Juniper Networks's routers
<a
href="http://www.eweek.com/c/a/IT-Infrastructure/Bug-in-Juniper-Router-Firmware-Update-Causes-Massive-Internet-Outage-709180/">caused
outages</a> in Level 3 Communications's networking backbone. This subsequently
knocked services like Time Warner Cable and RIM BlackBerry, and several UK
internet service providers offline.

### Global BGP outages

There have been several global Internet outages related to BGP
misconfiguration. Notably, in 2008, Pakistan Telecom, responding to a
government edict to block YouTube.com, incorrectly advertised its (blocked)
route to other provides, which hijacked traffic from the site and <a
href="http://news.cnet.com/8301-10784_3-9878655-7.html">briefly rendered it
unreachable</a>. In 2010, a group of Duke University researchers achieved
similar effect by <a
href="http://www.merit.edu/mail.archives/nanog/msg11505.html">testing</a> an
experimental flag in the BGP protocol. Similar incidents have occurred <a
href="http://www.renesys.com/2006/01/coned-steals-the-net/">in 2006</a>
(knocking sites like Martha Stewart Living and The New York Times offline), <a
href="http://www.renesys.com/2005/12/internetwide-nearcatastrophela/">in
2005</a> (where a misconfiguration in Turkey attempted in a redirect for the
*entire* internet), and <a
href="http://merit.edu/mail.archives/nanog/1997-04/msg00380.html">in 1997</a>.


## Where do we go from here?

This post is meant as a reference point, to illustrate that, according to a wide array of accounts, partition behavior occurs in many real-world deployments. Processes, servers, NICs, switches, networks, and globally distributed networks fail, and the economic consequences are real. Network outages can suddenly arise in systems that are stable for months at a time, or, alternatively, during routine upgrades or emergency maintenance. The consequences of these outages can range from increased latency and temporary unavailability to inconsistency, corruption, and data loss. Split-brain is not an academic concern: it happens to all kinds of systems--sometimes for *days on end*. Partitions deserve serious consideration.

In practice, some networks really *are* reliable. Engineers at major financial firms report that, despite putting serious effort into designing systems that gracefully tolerate partitions, their networks rarely, if ever, exhibit partition behavior. Moreover, cautious engineering and lots of money can prevent outages.

However, not all organizations can afford the cost or operational complexity of
highly reliable networks. And, from Google and Amazon, who operate commodity and/or low-cost hardware due to sheer scale, to one-man startups built on shoestring budgets, failures are a reality. It's important to consider partitions *before* they occur--because it's much easier to make decisions about behavior under partition on a whiteboard than to redesign, re-engineer, and upgrade a complex system in production environment, especially when it's throwing 404 errors at your users. For some applications, giving 404 is okay, but know when it's not, and plan for partitions.

Have an answer to the question: *when my servers can't talk, what happens to my application?* Based on what we've seen, your network is going to ask you.

*We invite and encourage you to contribute your own experiences with or without network partitions. Open a pull request on this page, leave a comment, write a blog post, or release a post-mortem. Data will inform this conversation, current practice, future designs, and, ultimately, the availability of the systems you depend on."