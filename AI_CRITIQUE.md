# AI Usage and Critique

Group: Edamame

Members:
- Lwin Oo (Edward, 6705140051)
- Kaung Myat Tun (Kingston, 6705140059)
- Ngyein Chan Ko (Reka, 6705140061)
- Han Htoo Aung (Erics, 6705140067)
- Wathan Htat (Frost, 6705140052)

## Part 3a: Declaration (required)

| AI tool | What we used it for |
| --- | --- |
| OpenAI Codex | Assistance with command syntax and unfamiliar commands during troubleshooting. After troubleshooting, Codex helped draft and revise the incident log using our session recording, check its evidence, review the submission requirements and repository structure, prepare the AI comparison prompt, and organize this critique. |
| Claude | Initially reviewed the completed session recording. For the comparison below, Claude also received the initial broken-lab symptoms and proposed possible causes, diagnostic commands, conditional repairs, and client verification steps. |
| ChatGPT | Project discussion and the diagnosis comparison below. ChatGPT received the initial broken-lab symptoms and proposed possible causes, diagnostic commands, conditional repairs, and client verification steps. |

The AI tools did not execute the new comparison's suggested commands on our lab. We evaluated their responses against our previously recorded troubleshooting, checker results, and the supplied lab scripts.

## Part 3b: Confirm you diagnosed first

We completed the recorded troubleshooting and incident log before conducting the symptom-based comparison below. AI assistance used during the project is declared above.

The four faults established during our lab work were:

1. The forward-zone declaration referenced `/etc/bind/db.MISSING` instead of the existing `/etc/bind/db.novapeak.lab`.
2. An INPUT firewall rule dropped UDP traffic to port 53.
3. The `api` A record contained `192.168.19.1` instead of `192.168.19.40`.
4. DHCP advertised `8.8.8.8` instead of the company DNS server, `192.168.19.10`.

The recorded checker score improved from 11.0/25 to 25.0/25.

Our earlier requests asking AI tools to review the completed session were retrospective reviews. The comparison below evaluates their responses to a separate prompt describing the initial broken state.

## Part 3c: Putting the AIs to the test

We gave Claude and ChatGPT the same initial observations:

- Ubuntu running BIND and ISC DHCP.
- Domain `novapeak.lab`, server interface `lan0`, and expected DNS server `192.168.19.10`.
- Expected A records: `ns1` at `.10`, `www` at `.20`, `mail` at `.30`, and `api` at `.40`.
- `named-checkconf -z` failed, while `named-checkzone` passed.
- DNS queries to the company server timed out.
- Forward lookups returned no answers, and the reverse lookup did not return `ns1`.
- The client obtained `192.168.19.104`; pool, mask, gateway, and company-domain checks passed.
- DHCP advertised `8.8.8.8`, and the client could not resolve `www` through its leased resolver.

We asked both tools to distinguish facts from hypotheses, propose diagnostic commands, recommend fixes after confirmation, explain client verification, and identify what remained unknown.

The prompt did not contain the missing filename, firewall listing, incorrect `api` record, or completed fixes. Therefore, we assessed whether their reasoning and proposed checks were justified by the available information. We did not expect them to know hidden configuration details.

### Comparison against the faults we actually proved

| Fault (in our words) | We found it? | AI #1 (Claude) | AI #2 (ChatGPT) | Was each AI actually right? |
| --- | --- | --- | --- | --- |
| BIND referenced a nonexistent forward-zone file. | Yes. `named-checkconf -z` reported `db.MISSING`; the zone declaration and file listing established the mismatch. | Explained the difference between checking an individual zone file and loading declared zones. Included a wrong file path among the possible causes, but also speculated about the reverse zone. | Recommended reading the exact configuration error and inspecting zone declarations and paths. Clearly stated that the precise cause was unknown. | Both proposed relevant checks. Neither identified `db.MISSING` from the initial symptoms alone. Our recorded output confirmed the forward-zone path error, and C4 passed after its correction. Claude's reverse-zone suspicion was not supported by our eventual findings. |
| A firewall rule blocked UDP DNS traffic. | Yes. The INPUT-chain listing showed a UDP port 53 DROP rule. Its counter increased from 7 to 8 after another query. | Included firewall filtering as a hypothesis and suggested inspecting UFW, nftables, and iptables, checking listeners, and capturing traffic. | Suggested comparing UDP and TCP queries, inspecting firewall rules, and capturing packets. Explained that a timeout alone does not identify its cause. | Both gave useful diagnostic directions, but neither had enough evidence to confirm the firewall rule initially. Our counter evidence and the successful client checker after removing the rule established the cause. |
| The `api` A record had the wrong address. | Yes. The zone file contained `192.168.19.1`; the checker expected `192.168.19.40`. | Correctly warned that passing zone validation does not prove record values are correct, and recommended inspecting the records. | Recommended establishing DNS reachability before deciding whether records were missing or wrong, then checking all four expected addresses. | Both were reasonable. Neither could establish the specific incorrect address from a timeout. Once reachability was restored, the checker exposed the wrong `api` answer. After the edit and BIND restart, C2 reported all four A records correct. |
| DHCP supplied the wrong resolver. | Yes. The lease result and `dhcpd.conf` both showed `8.8.8.8`. | Recommended advertising `192.168.19.10`, checking overrides, validating DHCP configuration, restarting ISC DHCP, and renewing the client lease. | Recommended the same resolver correction while distinguishing the observed wrong option from the still-unknown configuration source. | Both recommended the correct setting. Our recorded DHCP-service restart was followed by passing C9 and C11 checks. ChatGPT was more precise about what the initial observations proved: they showed the advertised value, but did not identify the responsible declaration. |

### Incorrect, misleading, or unverified suggestions

**1. A failed reverse lookup did not establish a separate reverse-zone fault.**

Claude suggested that a different zone, possibly the reverse zone, might explain the configuration-check failure. Both tools discussed reverse-zone repairs conditionally.

Our recording showed that the configuration error concerned the forward-zone path. After removing the firewall rule, C3 passed without editing the reverse zone. Therefore, reverse-zone modification was unnecessary in our actual repair. A conditional suggestion to investigate it was reasonable, but it should not be reported as a confirmed diagnosis.

**2. Claude's client-interface commands did not match this lab's topology.**

Claude used `lan0` in its client lease-renewal and resolver-inspection examples. During documentation review, we checked `setup_lab_net.py` and `check_lab.py`: `lan0` is the server-side interface, while the client uses `cli0` inside the `client` network namespace. The checker obtains its lease using `dhcpcd`.

ChatGPT used a `<client-interface>` placeholder and described `dhclient` as an example, which made its advice less specific but easier to adapt correctly. Neither answer provided the exact namespace-aware workflow used by this lab. The prompt did not explain that topology, so this is an environment assumption requiring verification.

**3. Generic client tests need adaptation to the lab.**

Both tools correctly emphasized testing through the DHCP-provided resolver. However, running a plain `dig` command in the server's terminal would not test the separate client.

The supplied checker executes DNS queries inside the client namespace and uses the DNS address obtained from its lease. We relied on the recorded C9 and C11 results as evidence that the advertised company resolver worked from that client. We did not claim that the AIs' suggested standalone client commands were newly executed.

**4. The proposed UFW rules were not our verified firewall repair.**

Both tools supplied examples of allowing DNS through UFW. Our evidence concerned a specific iptables INPUT DROP rule, and the recorded repair removed that identified rule.

We did not test the proposed UFW rules, so we cannot claim they would have repaired the existing rule arrangement. The relevant lesson is to inspect the active firewall and verify the effect of the chosen change.

**5. Additional changes should depend on the requirement and evidence.**

Claude suggested external forwarding as an alternative to giving clients a public resolver. External-name resolution was not the stated goal of this lab, and our submitted BIND options contain `recursion no;`.

We did not need external forwarding to achieve the recorded 25/25 result. We treated that suggestion as advice for a different requirement, rather than a necessary repair for this assignment.

## Part 3d: Verdict

**Where our human diagnosis added evidence beyond the AI hypotheses**

Our advantage was access to the running lab and its actual output. We established the missing forward-zone filename, observed the firewall counter increase, inspected the incorrect `api` record, and confirmed the wrong DHCP option. The AIs initially had only a symptom summary, so uncertainty about these details was appropriate. We should not mistake a longer list of possible causes for a completed diagnosis.

**Where AI helped**

Both tools provided a useful order of investigation: inspect configuration-loading errors, establish client reachability, verify record values, correct DHCP resolver settings, and test the complete client path. ChatGPT was generally careful about separating observed outcomes from unknown causes. Claude offered additional inspection ideas, including explicit iptables checks. Codex helped us organize the recorded evidence and evaluate whether the proposed commands matched this particular lab.

**What we learned about trusting AI to diagnose a live system**

AI advice needs to be checked against the actual interface names, network namespaces, services, firewall rules, and command output. We should apply a proposed fix only after establishing its cause, and verify it from the client's perspective. Our final 25/25 result supports the repairs recorded in `session.log`; it does not prove that every alternative command suggested by either AI would work. The strongest evaluation distinguishes confirmed findings, reasonable hypotheses, incorrect assumptions, and suggestions that remain untested.