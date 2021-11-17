# Viable Circumvention of State Internet Censorship

## Problem definition

State censorship of internet communications relies on various set of complex techniques and rules in order to isolate undesired traffic out of all communications and block it. In order to evade censorship, obviously, censored communication must obfuscate connection features which are subject of filtering. If connection is obfuscated and encrypted, it can incapsulate other traffic inside. So, solving this problem for VPN connections, we solve this problem for virtually all communications suffering from censorship.

It is practical to divide such connection features into two groups as their obfuscation require very different approaches:

* **Content features.** Features which can be learned from the content of connection. Examples: keywords, specific protocol patterns, statistical anomalies[^1] and so on.
* **Connection meta-information features**. Features which can be extracted from network information used to perform communication. Examples: network addresses (IPv4, IPv6), domain names in DNS requests or TLS server name indication, well-known port numbers of transport protocol, well-known IP protocol labels (EH/ESP for IPsec and other tunneling protocols over IP).

It is worth noting that censor does not need filtering criterias to be exactly accurate. It is acceptable for a censor to ban more than needed as far as there is not too much of collateral damage done. Because of it, cryptographical obfuscation of transfered data may be not sufficient alone if given cryptoprotocol is detectable by itself and mostly used for abuse of censorship filters.

Content features of communication is a *mostly* solved problem. There are many cryptographic connection wrappers around which try to mimic commodity protocols employing encryption by design (like TLS wrappers, SSH and so on). Motivated censor still can try to look closer at such connections and apply statistical analysis or active probing, though.

Meta-information features of communication, on the other hand, can't be easily obfuscated just by software operating the connection. It takes censorship-prevention solution to use network addresses and other related attributes unknown to censor as ones previously exposed in circumvention activities. For this reason it's quite easy to operate private censorship bypassing bridge, but rather challenging to achieve durable operation for a public service, especially for a service with open discovery of network relays: sooner or later addresses used by this service will be known as associated with it and banned. Even if circumvention service rotates addresses frequently, it doesn't appears to be a sustainable solution:

1. It works only under assumption there is a significant delay between rotation and update of filters which is not the case for modern theater of operations.
2. Usually costs for a censor to ban IP addresses much lower than for service operator to rotate addresses for entire server fleet. It allows censor to run attrition warfare against public censorship circumvention services.

Consider following situation: circumvention service operator wants to provide services to user, but censor wants to prevent it. Circumvention service operator releases application with some non-trivial logic to discover working relays and let users to pass their traffic through. All what censor needs to do is just use that application and keep track on IP addresses used to forward actual traffic. This way censor may ban all IP addresses learned until application will cease to work.[^2]

The ultimate task is to find a solution for meta-data obfuscation satisfying all requirements presented above.

## Related work

### Academic sources

Actual work doesn't claim novelty and for this reason the author skips detailed comparison against prior art to outline own contribution.

Internet censorship and anti-censorship have been [discussed by a great number of authors in literature](https://www.princeton.edu/~chiangm/anticensorship.pdf) over past decades. Some of these studies have valuable classification of methods used by state censors and had heavy influence on definition of problem for this work. However, no complete solutions were found which address concerning issues: some works are outdated by now, other successfully achieve advantage over censor in individual communications, but do not even consider challenges of circumvention service operation at scale, inherently not covering problems of meta-information exposure.

### Practical implementations

Analysis of several real-world systems, referenced below, also provided valuable input for this work.

[Tor Project](https://www.torproject.org/) is one of oldest projects in this field, which was remaining main anti-censorship tool until all its [guard nodes](https://community.torproject.org/relay/types-of-relays/) were banned by major actors. Project had multiple attempts to fix it with semi-private relays, ["bridges"](https://tb-manual.torproject.org/bridges/) disseminated among users various ways: [BridgeDB](https://bridges.torproject.org/), [Snowflake](https://snowflake.torproject.org/) and others. [By 2019 there were nearly 600 usable tor bridges](https://blog.torproject.org/run-tor-bridges-defend-open-internet), most of them are banned in Mainland China. Some [user reports](https://gitlab.torproject.org/tpo/anti-censorship/pluggable-transports/snowflake/-/issues/30350) suggest that Snowflake bridges (P2P bridges) are not available anymore too.

[Psiphon](https://www.psiphon3.com/en/index.html), an award-winning censorship circumvention tool, uses following techniques to establish communications (briefly):

1. Initial contact is established with central service using techniques like [domain fronting](https://attack.mitre.org/techniques/T1090/004/) to access such service. Censor cannot eavesdrop on communication content because it is protected by legitimate TLS session and cannot ban entire CDN service using meta-information only because of heavy collateral damage.
2. Application exchanges collected keys for proxies and receives access credentials and network addresses for proxy. Keys are disseminated across users by various methods: embedded keys in application build, social networks, obscure (kept in secret) rewards based on user behavior.

Several VPN providers, including Hola VPN[^3], use public 3rd party services as a cover for initial discovery of their application. For instance, Hola VPN publishes limited rotated list of proxies in public file on [Dropbox](https://www.dropbox.com/), allowing to access Hola API servers through these proxies and retrieve proxies and credentials for actual service.

[Amnezia VPN](https://en.amnezia.org/) is a free desktop application which allows user to setup own self-hosted VPN server. Supported protocols are UDP OpenVPN and TCP OpenVPN via shadowsocks, so decent connection obfuscation is available if shadowsocks option is used. If we consider cloud hosting providers in complex with Amnezia VPN application (or similar one) as a global public VPN service, we can notice following:

* Using state-of-art software it is possible to achieve content features obfuscation goal.
* Meta-information features (network addresses and so on) are not discoverable for censor because each VPN server is exposed to highly limited amount of customers. Also censor unlikely will ban all cloud hosting providers as a whole because of tremendous collateral damage associated with this action[^4].

Analysis of these solutions revealed some common key patterns:

1. Successful solutions (at least to some degree), in one way or another, always involve use of communication via services which can't be banned because of significant collateral damage associated with this ban. Also it is required that invididual interactions with such **cover service** cannot be isolated by censor determining their intention (e. g. cover service must use TLS to make censor unable to determine specific request is associated with censorship circumvention).
2. **Limited exposure** of connection meta-information (network addresses, domains and so on) is absolutely required. Public services failing to achieve this goal end up with most of servers banned or being involved in a losing game, rotating server addresses to have them banned again.

Having such measures in place, it is possible to construct viable state censorship circumvention solution.

## Proposed solution

Following these observations, actual proposal aims to generalize public censorship circumvention service with limited exposure of each entry point and provide additional guarantees desirable for trustless setting. This way we can solve meta-data exposure problem and allow untrusted participants to contribute into network with network resources available to them.

### Cover service

Observed solutions piggybacking well-known centralized services with significant social impact. It must be admitted that such move is a good ploy, but it is hardly a future-proof solution:

* Such techniques depend on specific properties of particular service, which may change over time.
* Service administration may eventually prevent such abuse.
* Major state actors encourage use of local services, lowering audience and social impact of foreign services over time. Once urge for censorship will outweight social value of entire cover service, it will be no longer a problem for the censor to ban such cover service.

Blockchain networks appear to be better alternative for this purpose because of following reasons:

* Decentralized. No central authority can prevent use of blockchain by censorship circumvention services.
* Hard to ban because each node joined blockchain network may become also an entrypoint for it. Even if banned, there are still many ways for a consumer to send transactions out of band.
* Already suited for value transfer transactions.
* Some blockchains offer rich capabilities to impose additional restrictions on transactions and that make blockchain a convenient framework to implement guarantees for setting with untrusted participants.

### Limited exposure

Limited metadata exposure requirement means each network requisite (IP addresses, CDN domain) to be exposed to a very limited number of distinct consumers. This way we can ensure low impact from censor learning address of some entry point.

Fixed upper boundary for address reuses of each entry point implies required number of entry point addresses proportional to audience. Consequently, as such addresses are not free to maintain resource, it imposes costs which has to be paid by consumer. On the other hand, costs for censor to learn all addresses will grow linearly with network size (if it manages to buy share on each entry point at all).

To achieve this, we can propose a system where untrusted providers obtain IP addresses or domains from source available for them (cloud hosting providers, domain registrars, CDN operators and so on) and resell access as a private bridge service for limited number of users.

### Service operation

#### NRD oracle

Network resource descriptor oracle is a key element of this system. It's a service which constructs network resource descriptor as follows:

```
deterministic_representation = "<ipv4|ipv6>:<address>|cdn:<domain>"
resource_signature = signature(deterministic_representation, NRD_oracle_privkey)
NRD = Keccak256(resource_signature)
```

where `NRD_oracle_privkey` is a private key of NRD oracle, which public counterpart is well-known by network participants. Domain has to be normalized to first significant component after public suffix: `www.some.domain.co.uk` => `domain.co.uk`. IPv6 address has to be normalized to /64 prefix containing it.

NRD for resource and `resource_signature` is handed by NRD oracle along with signed IP properties only to provider proved resource ownership. For IP address it can be done just by originating request to NRD oracle from IP address under the question. For domain names we can use challenges similar to [ones defined in ACME](https://letsencrypt.org/docs/challenge-types/).

NRD has following properties:

* NRD values for same resource are also equal.
* NRD can be matched against respective network resource only if `resource_signature` is supplied. This way it is possible for address owner to prove domain or IP address is matching some NRD value, but impossible to map this NRD to any IP address.

#### Provider registration workflow

1. Untrusted provider registers itself in smart contract on the blockchain by putting some minimal stake.
2. Provider setups entry point (bridge server) and claims ownership of associated entry point address. Ownership claim must contain recent enough proof from NRD oracle with its signature and details about this NRD (at least its kind: ipv4, ipv6, CDN domain).
3. Provider publishes proposal consisting of network resource descriptor, price and details about server (location country, IP class, ...), and number of reuses it intends for this entry point. Details about server are provided by NRD oracle and have its signature. It is required that NRD in proposal is already claimed by this provider.

#### Main workflow

1. Consumer fetches available proposals, validates them and picks proposal with some NRD. Only proposals from providers with minimal stake are considered valid.
2. Consumer generates public key pair to use for maintenance of agreement.
3. Consumer places an order containing following values: order UUID, maintenance pubkey, NRD, expected total number of server slots, number of slots paid by this order and duration of agreement. Also consumer deposits funds to cover payments into smart contract managing agreements.
4. Provider validates placed order and settles agreement by publishing following information to smart contract: order UUID, encrypted connection details. Connection details are encrypted with public key placed by consumer in its order. Connection details contain credentials required to make a tunnel connection with speficied server, server network address (IP or name of domain parked to CDN) and `resource_signature` from NRD oracle. After this transaction smart contract increases count of NRD usages by a number specified in order. Number of usages must not exceed minimal total number of usages across all agreements for this NRD.
5. Consumers retrieves latest encrypted connection details placed for its order and validates them. In case if connection details are incorrect, consumer may initiate refund workflow. Otherwise consumer may start using the bridge.
6. Provider may claim reward for this order only if it will supply set of periodical ownership proofs for this NRD covering time range up to agreement expiration. This way provider can receive reward only after agreement fulfilled and only if it owned network resource associated with NRD all the time. If provider will fail to perform this after some grace period, refund is performed.

Note that consumer may buy out all remaining slots of NRD, making this resource sealed against future use and have higher guarantees that no neighbour is cooperating with censor.

It is advisable to implement cooldown for NRDs which will allow new claims and new usages after some period of time. Following cooldown periods can be suggested[^5]:

|  NRD Kind  | Cooldown Period |
|  :------:  | :-------------: |
|    IPv4    |      1 year     |
|    IPv6    |      1 year     |
| CDN Domain |     10 years    |

#### Connection information update

Provider may assign new encrypted connection information and NRD for agreements. Same rules apply as for initial assignment. It is useful for following pusposes:

* Move consumers out of banned server to new different NRDs of same kind. Even if some user causing problems, with each iteration lesser amount of users will be affected until only one or few problematic users will remain.
* Compaction. If most of consumers attached for some server stopped paying for it, it is advantageous for a provider to relocate remaining consumers of several network resources to one new resource and reduce costs of rented unpopulated servers.
* Maintenance or migration.

#### Renewal workflow

Same as main workflow, but reusing existing order UUID and setting new expiration time for it. Renewed agreements do not use additional slots of NRD. Optionally, lower price may be allowed by provider.

#### Refund workflow

Refund workflow is a measure which is supposed to guard consumers against dishonest providers which failing to prove supplied network resource indeed matches NRD announced in proposal or supply undecryptable ciphertexts or do not provide service after purchase.

1. Consumer sends order UUID to the refund service with private key used for order.
2. Refund service retrieves latest connection details for that agreement and tries to decrypt connection details. If private key doesn't match public key used for agreement, request gets rejected. If private key matches, but decryption failed, refund for this order is initiated immediately and processing stops with successful result.
3. If provided connection info doesn't contain valid proof of match of network resource address to NRD, refund for this order is initiated immediately and processing stops with successful result.
4. Refund service performs few attempts to use the bridge with exponential back-offs. If all attempts failed, refund is initiated.

Refund transactions are paid at the expense of stake of provider served that order. Optionally, it is possible to apply additional fine to stake of that provider.

## Relation to Mysterium Network

Proposed solution is quite the opposite of Mysterium Network in many aspects:

* Mysterium is a public system which tries to maximise number of available exit nodes for a consumer. Proposed solution is restrictive and minimizes number of consumers knowing about each node.
* Mysterium has pay-as-you-go basis while proposed solution is strictly prepaid as it is critical for its sustainability.
* In Mysterium provider is just a single instance of running node, while proposed notion of provider is actual provider of multiple network resources and proposals associated with them.
* Mysterium favors residential nodes, but for this proposal datacenter nodes is more favorable:
  * Datacenter nodes have higher SLA.
  * Datacenter addresses, especially hosted by major cloud providers are more valuable in regard of massive network bans conducted by censor. Ban of multiple residential networks of other countries will go mostly unnoticed when ban of few cloud providers will have massive collateral damage.
* Mysterium network allows dynamic presence of network members while current proposal aims for proposals having high SLA. It is a consequence of limited alternatives revealed for a single consumer.
* Mysterium network uses Wireguard as its tunnel protocol, but it can be detected and banned by DPI. For this proposal we suggest protocols indistinguishable from legitimate connections.
* Mysterium network relies on centralized infrastructure to allow consumer perform transactions on the blockchain. This proposal aims for direct use of blockchain by consumers.

Actual proposal relates to Mysterium network in the same way as a paid private bridge service relates to Tor network: a private extension for a public network. It is supposed that it will be a separate entity augmenting Mysterium Network and allowing censored users into network. But also it is possible to market such solution as a separate self-sufficient VPN service.

[^1]: One remarkable example is a detection of elliptic curve public keys. Elliptic curve public key is represented by a coordinate on elliptic curve over finite field. Some cryptosystems use compressed form and recover point on curve from just one coordinate component. Because of elliptic curve definition, this number is always some square number over chosen modulus (which is fixed for curve and cryptosystem built on it). Therefore in all cases such number can be positively tested as a quadratic residue over that modulus (i.e. having 100% positive rate). Arbitrary random number has 50% chance to be quadratic residue over prime modulus field (which is used for most ECC schemes). This way having 100% successful test over numbers extracted from many observed connections it is possible to conclude that it's not just a random data, but an elliptic curve public key, despite the key itself was generated randomly. There are some [techniques](https://elligator.cr.yp.to/) around to hide that particular anomaly, but there are also other options for censor to make statistical assumptions with acceptable accuracy.
[^2]: That actually happened in Russia in 2018, but not with a VPN application, rather with Telegram instant messenger instead. ROSKOMNADZOR engineers automated the process using mobile application and WiFi router with OpenWRT firmware to track exposed IP addresses. Their attempts of Telegram ban mostly failed because of two reasons: significant delay between blacklist updates (ISPs obligated to update it once in one or few days) and Telegram having side channel to push updates with new IP addresses through push messages via Apple, Google and Microsoft services. Modern landscape rapidly changes towards  direction infavorable for censored services: ISPs getting more centralized, state bodies now have direct control over dedicated filtration equipment (for lower blacklist lag) and large IT companies are presumably lobbying standards to shift such debatable communications to an application infrastructure in order to make it blockable without collateral damage to their global shared services: [RFC 8765](https://datatracker.ietf.org/doc/html/rfc8765).
[^3]: Author also provides [open-source alternative implementation](https://github.com/Snawoot/hola-proxy/) of Hola browser extension in form of standalone application, improved to work in Egypt by hiding TLS server name indication in its connections.
[^4]: Such massive bans of cloud hosting providers actually happened in Russia in 2018 on desperate attempts to block Telegram proxies in advance, but in general state won't be able to sustain such strategy for long because of two reasons: (1) economical damage; (2) growing incentive to seek for solutions even among population not interested in circumvention, effectively shifting balance of force against state censor.
[^5]: Usually observed cooldown for IPv4 addresses in Chinese GFW is 2-3 months, so 1 year is a safe value. Domain space is practically infinite, so there is no reason for a censor to unban domain and not much reasons for a provider to reuse it.
