Footnote
=======================

Footnote is an experimental Handshake layer two protocol for storing,
synchronizing, and addressing small amounts of data. Its purpose is to create a
global, decentralized, and permissionless data commons that allows everyone to
store and retrieve data from any node on the Footnote network.

Abstract and Motivation
-----------------------

The Internet today favors a client-server model. Clients - who function like
customers - request or upload data to servers - who function like vast
warehouses of data under the control of the companies that own them. For most
applications, this works well. There's no need nor ability for one's browser to
know everything about every other Facebook user, the same way there's no need
for people to stockpile every item at their local Costco.

The client-server model, however, has its tradeoffs. Take Twitter: whenever a
user sends a Tweet, that Tweet becomes the property of Twitter.  Similarly, it
is not Twitter's users that decide what appears on their news feeds but rather
Twitter itself. Oftentimes this is done for scalability or content moderation
reasons. Twitter needs to serve Tweets to over three hundred million people, so
unfettered access to their firehose would be technically infeasible. Similarly,
while ads help Twitter pay for hosting, not all content is advertiser-friendly.
In sum, clients sacrifice the sovereignty of their data, as well as their
ability to query other people's data, in return for scale and convenience.

Footnote exists to serve data that may not fit the existing client-server mold.
Rather than building another warehouse, Footnote creates an open plaza that
anyone can observe or contribute to.  Every node on the Footnote network serves
everyone else's data. There's no DHT querying, sharding, or partitioning: every
node is equal. This allows developers to build rich, interconnected applications
on top of Footnote that leverage a persistent, global view of all content on the
network.  Imagine a version of the Internet built to be "locked open"; every
server stores every website's content by default.  While this might sound
redundant at first, consider the decentralization-preserving protective effects
(i.e. equal access) if the web we know today was built on top of it.  Users
could query data locally by default, and be able to choose which filters or and
moderation to apply (or even to supplement with centralized infrastructure).
This is what Footnote aims to enable.

Creating such a global view would be impossible without constraints.  This is
because without constraints, as the network grows, resource and administration
requirements increase.  This effect prices out smaller players.  Footnote's
constraints are (i) bounded storage available to (ii) a bounded number of names.
For the former, Footnote identifies users by their Handshake name, allowing each
user to store a maximum of 1 mebibyte on the network.  For the latter, Handshake
itself imposes a maximum global renewal limit through its on-chain transaction
throughput: around 50-100 million names can exist on Footnote. Taken together, a
Footnote node's maximum storage requirement is on the order of tens of
terabytes. As the cost of storage trends downwards, the resources required to
run a Footnote node should trend downwards as well, even as the network itself
grows in usage.  In sum, Footnote leverages Handshake's Sybil-resistance
properties and constraints to create the first durably decentralized public data
commons. 

Protocol Specification 
-----------------------

At this time, there is no canonical specification.  Early in the design process,
a loose protocol specification was previously defined through a series of
Protocol Improvement Proposals (PIPs), which can be found
[here](https://github.com/kyokan/footnote-PIPs).  They are made available for
accuracy and to provide context.  Please see the other document in this
repository, [ideas.md](https://github.com/kyokan/footnote/blob/master/ideas.md)
for examples of potential protocol improvements.


Links
-----

#### Protocol Implementations

| Language | Project | Notes |
| ----- | ---- | ----- |
| Go | [fnd](https://github.com/kyokan/fnd) | Reference Implementation, by [@mslipper](https://github.com/mslipper) |

#### Client Libraries

| Language | Project | Description |
| ----- | ---- | ----- |
| TypeScript | [fn-client](https://github.com/kyokan/fn-client) | RPC client for fnd, message format utilities, and [social](https://github.com/kyokan/footnote-PIPs/blob/master/pip-008.rst) records |
