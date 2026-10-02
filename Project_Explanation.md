# Firewall Rule Design and Network Segmentation Concept for a Bank Using Site-to-Site VPN

**Author:** Meleng — IT Intern, Bridge Technologies Solutions (Douala, Cameroon)
**Tools:** GNS3, VMware Workstation Pro, FortiGate (FortiOS 7.0.14), Cisco IOS 12.4, Docker (Alpine Linux)

This document is a complete, plain-language walkthrough of the project — what it is, why it's built the way it is, how it was tested, and what went wrong along the way.

---

## 1. The problem this project solves

A bank with a Head Office and a Branch needs two things at once:

1. **Internal separation** staff, servers, databases, ATMs, and guests should not all sit on one flat network. If one device is compromised, the damage should stay contained.
2. **Secure communication between sites** Head Office and Branch need to share application and transaction traffic over a link neither of them physically controls (the Internet), without that traffic being readable or alterable in transit.

This project builds both, in a simulated lab, and proves not just claims that they work.

## 2. What was actually built

- Two **FortiGate firewalls** (FGT-HQ, FGT-BR), each acting as the default gateway for every VLAN at its site.
- **Seven VLANs at HQ** (management, staff, application servers, database, ATM, DMZ, guest) and **three at the Branch** (management, staff, ATM).
- A **site-to-site IPsec VPN** (route-based, IKEv2) directly between the two FortiGates, carrying only the traffic that's explicitly allowed.
- **Least-privilege firewall policies** — every flow between zones has to be named in a rule before it's allowed; everything else is dropped by default.
- A simulated Internet (a Cisco router) between the two sites, and Ethernet switches carrying tagged VLANs to each host.
- Later, a small **web front end** on the application server (a login page and account dashboard) and **browser access to both firewalls' management GUIs**, so the project isn't only testable from the command line.

## 3. Why the firewall is the gateway for every VLAN

This is the single most important design decision. If a router did the inter-VLAN routing instead, traffic moving between VLANs would never pass through the firewall at all — routers just forward packets, they don't apply security policy. By making the FortiGate the default gateway for every VLAN, **every packet that crosses from one zone to another is inspected against a rule.** That's what turns "VLANs" into "segmentation" — the separation is enforced, not just organizational.

## 4. Why each zone is separate

| Zone | Why it's isolated |
|---|---|
| Management | The only network allowed to administer the firewall and infrastructure — reduces the attack surface for the most powerful access level. |
| Staff | The largest source of human error and malware exposure — must never reach the database directly. |
| Application servers | Sits between users and data on purpose, so business logic — not raw data access — is what's exposed. |
| Database | The highest-value target. Only the application tier may connect to it, and only on the database port. |
| ATM | Physically exposed in public places, so it's treated as more likely to be tampered with. |
| DMZ | Faces the Internet, so it must never be a stepping stone into the internal network even if it's compromised. |
| Guest | Untrusted by definition — gets Internet access and nothing else. |

## 5. How the VPN works, in plain terms

A site-to-site VPN joins two networks, not two users. The two firewalls are the tunnel's endpoints; it's always available, and no one has to log into a VPN client. That's the right shape for two bank sites that constantly need to talk to each other, as opposed to a remote-access VPN, which is built for one traveling employee connecting in after authenticating personally.

IPsec builds the tunnel in two phases:

- **Phase 1 (IKE):** the two firewalls prove who they are to each other — here, using a shared secret (pre-shared key) — and agree on encryption settings, creating a secure channel for negotiation itself.
- **Phase 2:** using that secure channel, they agree on the actual keys that will encrypt real traffic, and state exactly which subnets the tunnel protects (called selectors).

This lab uses a **route-based** VPN: the tunnel is a virtual interface with its own IP address, so ordinary routing decides what uses it, and ordinary firewall policies control it — the same way any other interface is controlled. That's more flexible and more consistent with the rest of the design than the older "policy-based" style, where the encryption decision is baked directly into a rule.

## 6. How the rules were written

Every rule follows the same pattern: **source zone, destination zone, allowed service, and a stated reason.** For example, Staff can reach the application servers over HTTPS, because that's how they use the banking application — but Staff has no rule to the database at all, because there is no legitimate reason for a user's PC to talk to a database directly. Only the application servers have a rule to the database, and only on the database's port.

The same logic applies across the VPN. Branch staff can reach the HQ application servers, because that's the shared banking application — but Branch staff has no rule to the HQ database, for exactly the same reason as local staff. **Nothing was ever allowed "Any → Any."** Early in building the VPN, a pair of allow-everything rules were used briefly just to prove the tunnel itself could negotiate — they were deleted and replaced with specific rules the moment that was confirmed, because a broad rule on a VPN tunnel turns two segmented networks into one flat one.

## 7. How it was tested — not just configured

Configuration alone doesn't prove a firewall works; only traffic does. Testing happened in layers:

1. **Basic reachability** — every firewall could ping the simulated Internet router, and every host could reach its own gateway.
2. **Segmentation, with ping** — thirteen tests covering every meaningful zone pair, both inside HQ and across the VPN. Each test recorded whether the flow was supposed to succeed or fail, and it matched every time: allowed flows got replies, denied flows timed out.
3. **Segmentation, with real application traffic** — ping alone doesn't prove an HTTPS rule works, since ping and HTTPS are different services. A small Python HTTPS server was built on the application server, and a real client fetched a real page from it — first locally, then from the Branch, across the encrypted tunnel. A request to the database VLAN from the same Branch client was correctly refused, proving the rule is protocol-aware, not just address-aware.
4. **Log and counter evidence** — the firewall's own session table and traffic logs were checked directly. Allowed traffic showed up with the matching policy name and, for VPN traffic, was explicitly tagged as IPsec traffic. Denied traffic created no session at all — which is itself proof that it never got through.

## 8. What went wrong, and what that taught me

Almost every real lesson in this project came from something breaking, not from something working on the first try. A few of the more important ones:

- **FortiOS requires an explicit VDOM setting** on new interfaces even with VDOMs disabled — an error message that looked obscure at first turned out to be exactly literal once read carefully.
- **A route-based VPN tunnel interface needs a /32 mask**, not a shared subnet between the two ends — each side is an independent host address.
- **FortiOS won't even attempt to negotiate a VPN tunnel until a firewall policy references the tunnel interface** — before that, it treats the traffic as having "no policy configured" and doesn't bother trying.
- **The evaluation license caps the firewall at 10 policies per VDOM**, which forced real design decisions about which flows could be merged into one rule without losing clarity.
- **The single most instructive bug**: giving a firewall's management interface a DHCP address (to make the admin GUI reachable from a real browser) caused FortiOS to silently install a *second* default route learned from that DHCP lease — and because DHCP-learned routes get a better administrative distance than a manually configured static route, the firewall's entire default route silently switched away from the real WAN link. This broke the VPN's ability to reach the other site at all, with no obvious error pointing at the cause — it had to be found by reading the routing table directly and noticing the default route pointed somewhere unexpected. The fix was two-fold: raise the static default route's own distance so it wins again, and explicitly disable `defaultgw` on the DHCP-configured interface so it never contests the default route in the first place.


## 9. Limitations, stated honestly

- The available FortiGate image only supports DES-based IPsec proposals, not AES — DES is obsolete and this was flagged and approved by my supervisor as a lab constraint, not a real recommendation. A production tunnel should use AES-256.
- ICMP was added to every policy purely so the lab's simple test hosts could exercise the rules; a production ruleset would remove it except where genuinely needed for monitoring.
- The firewalls' admin HTTPS service has a bug on this image (the TLS handshake resets before completing) — the admin GUI is only reachable over plain HTTP, which is not something a real deployment should ever do. This is documented as an image defect, not a design choice.
- Logging lives in memory only, since there's no log disk attached, so it doesn't survive a reboot.

*This document accompanies the full project report (topology diagrams, IP addressing plan, complete firewall rule tables, and detailed test results) and the GNS3 lab files in this repository.*
