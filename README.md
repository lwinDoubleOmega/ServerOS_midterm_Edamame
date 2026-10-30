# Server OS Midterm Submission

- **Group:** Edamame
- **Company / domain:** NovaPeak / `novapeak.lab`
- **Group ID used with the checker:** `lab01`
- **Members:**
  - Lwin Oo (6705140051) — GitHub: lwinDoubleOmega
  - Kaung Myat Tun (6705140059) — GitHub: Kaungmyattun8
  - Ngyein Chan Ko (6705140061) — GitHub: Reka1X
  - Han Htoo Aung (6705140067) — GitHub: Erics12487254
  - Wathan Htat (6705140052) — GitHub: callmefrost27-dotcom

## Submission files

- `fixed-configs/` — repaired DNS and DHCP configuration files
- `INCIDENT_LOG.md` — incident evidence, diagnosis, and fixes
- `AI_CRITIQUE.md` — AI use declaration, comparison, and verdict
- `session.log` — recorded troubleshooting session
- `checker_result.txt` — saved output from the final checker run

## Fixed configuration files

The `fixed-configs/` directory should contain:

- `named.conf.options`
- `named.conf.local`
- `db.novapeak.lab`
- `db.novapeak.lab.rev`
- `dhcpd.conf`
- `isc-dhcp-server`

## Checker result

Run the checker and save its output in `checker_result.txt`:

\`\`\`bash
python3 /root/grading/check_lab.py --group lab01 --phase build | tee checker_result.txt
\`\`\`