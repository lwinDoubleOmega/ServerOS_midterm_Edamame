# AI Usage and Critique

Group: Edamame   

Members: 
- Lwin Oo (6705140051)
- Kaung Myat Tun (6705140059)
- Ngyein Chan Ko (6705140061)
- Han Htoo Aung (6705140067)
- Wathan Htat (6705140052)

## Part 3a: Declaration (required)

| AI tool | What we used it for |
| --- | --- |
| OpenAI Codex | During Part 1, we mostly used Codex for help with mistyped commands, command syntax, and commands we were unfamiliar with. Our group ran the diagnostics, identified the faults from the results, chose the fixes, and verified them. After troubleshooting, we also used Codex to review the session log and help organize the evidence and documentation. |
| Claude | After our troubleshooting, we asked it to review the session log, explain the faults and evidence, and suggest further checks. |

## Part 3b: Confirm you diagnosed first

We completed the troubleshooting and prepared our incident log before the formal AI comparison in Part 3c. During Part 1, Codex mainly helped with command syntax and unfamiliar commands. Our fault diagnosis came from the commands we ran, their output, and the checker results.

The faults we identified were:

1. BIND’s forward zone declaration pointed to a nonexistent file.
2. A firewall rule dropped inbound UDP DNS traffic.
3. The `api` A record had the wrong IP address.
4. DHCP advertised `8.8.8.8` instead of the company DNS server.

## Part 3c: Putting the AIs to the test

During Part 1, we used Codex mainly for command syntax and unfamiliar commands. After troubleshooting, we gave Codex and Claude the session log and asked them to explain the faults and evidence and recommend fixes or further checks. We compared their explanations with the commands and checker results in the log.

| Fault (in our words) | We found it? | AI #1 (Codex) | AI #2 (Claude) | Was each AI actually right? |
| --- | --- | --- | --- | --- |
| BIND pointed the forward zone at a missing file. | Yes. `named-checkconf -z` reported that `/etc/bind/db.MISSING` was missing, and `ls` confirmed it. | Identified the incorrect zone-file path and the existing zone file to use. | Identified the incorrect path and noted that the real zone file existed. | Both identified the cause correctly. The checker confirmed that the zone loaded after the path was fixed. Claude’s explanation based on the file-size change was an inference, not direct proof of the edit. |
| The client timed out reaching DNS. | Yes. The firewall listing showed an inbound UDP port 53 DROP rule, which was removed. | Identified the firewall rule as the cause of the timeout. | Identified the UDP DNS drop as a separate fault and discussed the TCP query. | Both correctly identified the UDP firewall fault. Claude’s explanation of the earlier TCP result depends on the recorded command being mistyped; the terminal recording is unclear, so we could not fully verify that detail. |
| The `api` DNS record returned the wrong address. | Yes. The zone file showed `192.168.19.1`; the checker expected `192.168.19.40`. | Identified the incorrect `api` A record. | Identified the same incorrect record and noted that the missing-zone fault had hidden it at first. | Both were correct. After the zone loaded, the checker showed the wrong address; after the record was fixed, it reported all four A records correct. |
| DHCP advertised a public DNS server. | Yes. The config contained `option domain-name-servers 8.8.8.8;`, and the checker reported that value. | Identified that DHCP should advertise `192.168.19.10`. | Identified the incorrect resolver and noted that the DHCP service needed to be restarted for the change to take effect. | Both were correct. The final checker run confirmed that DHCP advertised the company DNS and that the client resolved `www` through it. |

Claude also said a rogue DHCP server was “almost certainly” ruled out. We compared that statement with our session log: it confirms the client’s lease and DNS resolution, but does not establish whether another DHCP server was present. We therefore treated Claude’s claim as unverified.

## Part 3d: Verdict

- **Where our human diagnosis beat the AIs:** During Part 1, our group used the checker and diagnostic commands to find and fix the network faults, then confirmed the results. Codex helped us with command syntax and unfamiliar commands. After troubleshooting, we asked Codex and Claude to review the session log and explain how its evidence showed the faults and fixes..
- **Where an AI genuinely helped:** Codex helped us correct mistyped commands and understand unfamiliar commands during Part 1. Later, Codex and Claude helped explain and organize the evidence in our session log for the documentation.
- **What we learned about trusting AI to diagnose a live system it cannot see:** AI suggestions can help guide troubleshooting, but we need to check them against actual command output and the checker. We should distinguish what the evidence proves from what an AI is inferring.