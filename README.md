# IPSec-Policy-Configuration-Verification-Lab
This lab followed a guided CompTIA exercise structure. The steps and objectives were provided; the configuration, troubleshooting, and Wireshark verification below are my own work.

# IPSec Policy Configuration & Verification Lab (Windows Local Security Policy, Wireshark, ISAKMP/IKE Analysis)

`IPSec` · `Windows Local Security Policy` · `Wireshark` · `ISAKMP/IKE` · `PowerShell` · `DVWA` · `CompTIA Labs`

## Overview
This lab was hands-on practice **configuring host-to-host IPSec encryption** using the Windows Local Security Policy console, then using Wireshark to prove — packet by packet — that the policy actually changed what was going out on the wire. I built a custom IPSec policy from scratch (IP filter list, filter action, security methods, authentication method), assigned it between two lab hosts, and then captured the resulting IKE negotiation to confirm it worked, rather than just trusting the GUI said "Yes" under Policy Assigned.

I also used an unencrypted DVWA web session earlier in the same session as a deliberate contrast: plain HTTP in Wireshark shows everything — cookies, session IDs, full page content — in cleartext, which is the exact problem IPSec is meant to solve for host-to-host traffic.

## Objective
Get comfortable building a custom IPSec policy end-to-end in `secpol.msc` (filter lists, filter actions, security methods, authentication), assign it between two hosts, and confirm success independently in Wireshark by reading the actual IKE Phase 1 / Phase 2 negotiation rather than relying on the console alone.

## Environment
- **PC10-v2023-06 (10 IPSEC VM10):** `10.1.24.101` — primary Windows host; where the IPSec policy was built, assigned, and where Wireshark captures were taken
- **PC20:** `10.1.24.102` — secondary Windows host; ping/policy target
- **Gateway:** `10.1.24.254`
- **Unreachable internal host (troubleshooting target):** `10.1.16.242`
- **DVWA web application:** `dvwa.structureality.com` (`172.16.0.201`) — used to generate a plaintext HTTP baseline
- **Lab platform:** CompTIA Learning Platform, hosted lab environment via LabClient (labondemand.com)

## Tools I Used

| Tool | What It Does | Why I Used It |
|------|--------------|----------------|
| **Local Security Policy (secpol.msc)** | Windows policy console | Built the custom IP Security Policy: filter list, filter action, security methods, and authentication method |
| **Wireshark** | Packet capture/analysis | Verified the IKE handshake (ISAKMP Main Mode + Quick Mode) actually occurred after assigning the policy, and captured a plaintext HTTP session as a baseline comparison |
| **PowerShell / Command Prompt** | Command-line testing | `ping` to confirm host reachability before and after the policy was applied; `curl` (Invoke-WebRequest) against DVWA to generate a clean HTTP request for the plaintext baseline |
| **DVWA** | Deliberately vulnerable web app | Source of plaintext HTTP traffic (cookies, session IDs, page content) used to contrast against encrypted IPSec traffic |

## What I Did

### Establishing a Baseline
1. Pinged `10.1.24.101`, `10.1.24.102`, and the gateway `10.1.24.254` from the command line to confirm all three were reachable before touching any policy, and captured the exchange in Wireshark to see plain ICMP Echo Request/Reply in the clear.
2. While testing reachability to `10.1.16.242`, repeatedly saw ICMP **Destination Unreachable (Host Unreachable)** messages coming back from the gateway (`10.1.24.254`). Filtered with `icmp and icmp.type!=3` to strip out those unreachable messages and isolate any actual echo traffic, which confirmed the host was genuinely unreachable rather than a capture/filter problem on my end.
3. Ran `curl http://dvwa.structureality.com` and browsed to the DVWA instructions page, then filtered Wireshark on `http` to look at the full unencrypted exchange — GET requests, `Set-Cookie: security=low`, `PHPSESSID`, server headers, and full HTML content, all readable in plaintext. Used this as the "before" reference point for what IPSec is meant to hide.

### Building the Custom IPSec Policy
1. Opened **Local Security Policy → IP Security Policies on Local Computer → Create IP Security Policy**, naming it `Structureality IPSec Policy`.
2. Built the security rule's **IP Filter List** with a destination address of **Any IP Address**, so the rule would apply broadly rather than to one narrow host pair.
3. Created a custom **Filter Action**. My first pass was named `Encrypt Some of The Things` — going back through it, I wasn't confident it actually forced encryption on all matched traffic, so I rebuilt it as `Encrypt All The Things` to make sure the action required negotiated security rather than leaving it optional/negotiable-but-not-mandatory.
4. Configured the **Security Methods** for that filter action as a custom method pairing **AH Integrity: SHA1** with **ESP Confidentiality: 3DES** (rather than accepting the default single option), giving the policy both integrity and confidentiality protection instead of just one.
5. Set the **Authentication Method** to a **preshared key** (`Password!Password!`) instead of the Active Directory/Kerberos default or a certificate authority, since these two lab hosts weren't domain-joined and a CA wasn't set up — the appropriate choice for this specific pairing rather than the "ideal" production choice.
6. Right-clicked the finished policy and selected **Assign** to make it active, confirming the **Policy Assigned** column flipped from `No` to `Yes`.

### Verifying the Policy Actually Worked
1. Rather than trusting the console's "Yes," opened Wireshark and filtered on `isakmp and (ip.addr==10.1.24.101 and ip.addr==10.1.24.102)`.
2. Saw the full **IKE negotiation** play out: **Identity Protection (Main Mode)** packets going back and forth to negotiate identity and keying material, followed by **Quick Mode** packets negotiating the actual IPSec security association — direct proof the two hosts negotiated an encrypted tunnel rather than just silently passing traffic.
3. Re-ran `ping 10.1.24.102` after assignment to confirm traffic between the hosts still succeeded, now under IPSec protection rather than in the clear.

## What's in This Repo

```
ipsec-policy-configuration-lab/
├── README.md                          # This file
├── screenshots/
│   ├── 01-baseline-ping-tests.png
│   ├── 02-icmp-unreachable-filter.png
│   ├── 03-dvwa-http-plaintext-capture.png
│   ├── 04-create-ip-security-policy.png
│   ├── 05-filter-action-name.png
│   ├── 06-security-methods-ah-esp.png
│   ├── 07-authentication-preshared-key.png
│   ├── 08-ip-filter-destination.png
│   ├── 09-assign-policy.png
│   └── 10-isakmp-handshake-wireshark.png
```

## Skills I Picked Up
- **Building an IPSec policy from its individual parts,** rather than clicking through a wizard on autopilot: filter list, filter action, security methods, and authentication method each configure a different part of the negotiation, and understanding what each one controls made troubleshooting far easier.
- **Not trusting a GUI's status column at face value.** "Policy Assigned: Yes" only tells you the policy is active, not that it's actually negotiating correctly — Wireshark was the real proof.
- **Reading an IKE handshake in Wireshark,** distinguishing Main Mode (Phase 1 — identity and key negotiation) from Quick Mode (Phase 2 — the actual security association) instead of treating "ISAKMP traffic" as one undifferentiated blob.
- **Isolating signal from noise in a busy capture,** using `icmp.type!=3` to filter out repeated Destination Unreachable messages so I could see whether any real echo traffic was underneath them.
- **Seeing plaintext exposure firsthand,** rather than just being told HTTP is insecure — watching session cookies and full page content scroll by in cleartext in the DVWA capture made the case for encryption concrete instead of abstract.
- **Revising a configuration when I wasn't confident in it,** rebuilding the filter action from "Encrypt Some of The Things" to "Encrypt All The Things" once I wasn't sure the first version actually enforced encryption, rather than leaving ambiguous config in place.

## How This Applies in the Real World
Host-to-host IPSec is exactly how sensitive internal (east-west) traffic gets protected inside a segmented network — between a domain controller and a file server, for example, or between two hosts handling regulated data where TLS isn't already in place at the application layer. Being able to read an IKE handshake in Wireshark is a core troubleshooting skill any time a site-to-site VPN or host-to-host IPSec tunnel fails to come up: seeing where the negotiation stalls (Main Mode never completing, Quick Mode rejected) tells you immediately whether the problem is authentication, proposed security methods mismatching, or something else entirely.

The preshared key I used here is a reasonable choice for a small, non-domain lab pairing, but I'm aware it doesn't scale — production environments manage this with Kerberos (inside a domain) or certificates issued by an internal CA, precisely because a shared static string like `Password!Password!` becomes a liability the moment more than a couple of hosts need it.

## Where I'm Coming From
I'm making the jump into cybersecurity from a background in **healthcare**. It's a different field on paper, but a lot of the muscle memory carries over: following procedures carefully, protecting sensitive information, staying calm and methodical when something isn't working the way it's supposed to. I'm currently studying for **CompTIA Security+** and building labs like this one to get real hands-on reps in, since that's what I'm missing on paper right now compared to my experience.

I left the first filter action attempt (`Encrypt Some of The Things`) in the writeup rather than cleaning it out, because rebuilding it once I wasn't confident it was correct is part of the actual process, not a mistake to hide.

## What I Want to Learn Next
- Inspecting the actual ESP-encrypted payload in Wireshark to confirm the application data itself is unreadable, not just that a negotiation occurred
- Rebuilding this same policy using a certificate authority instead of a preshared key, to compare the authentication flow
- Configuring IPSec via `netsh advfirewall` or PowerShell cmdlets instead of the GUI, for a repeatable/scriptable version of this same policy
- Testing a deliberate mismatch (e.g., differing security methods on each host) to see what a failed negotiation looks like in Wireshark

## Limitations & What I'd Do Differently in Production
- **Preshared key authentication doesn't scale.** A real deployment would use Kerberos (within a domain) or a certificate authority — a shared static key across more than two hosts becomes a real liability.
- **Only one host pair was tested.** A production rollout would need to validate the policy across many hosts and confirm it doesn't break unrelated traffic matched by the "Any IP Address" filter.
- **I verified the negotiation, not the payload.** Confirming ISAKMP/IKE completed successfully is good evidence but isn't the same as inspecting the encrypted ESP traffic itself to confirm confidentiality end-to-end.
- **No failure-mode testing.** I didn't intentionally break the configuration to see what a mismatched or rejected negotiation looks like, which would be valuable for real troubleshooting readiness.

## References
- [Microsoft: IPSec Policy Overview](https://learn.microsoft.com/en-us/windows-server/networking/technologies/networking)
- [Microsoft: IP Security Policies Snap-in](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/appendix-l--configuring-ipsec-policies)
- [Wireshark ISAKMP/IKE Wiki](https://wiki.wireshark.org/ISAKMP)
- [CompTIA Security+ (SY0-701) Exam Objectives](https://www.comptia.org/certifications/security)
