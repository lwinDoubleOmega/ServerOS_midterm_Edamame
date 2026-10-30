# Incident Log

Group: Edamame  

Members: 
- Lwin Oo (Edward, 6705140051)
- Kaung Myat Tun (Kingston, 6705140059)
- Ngyein Chan Ko (Reka, 6705140061)
- Han Htoo Aung (Erics, 6705140067)
- Wathan Htat (Frost, 6705140052) 

Company / domain: NovaPeak / `novapeak.lab`

Evidence below consists of selected command-output excerpts from `session.log`; omitted output is available in the original recording. Terminal cursor and editing codes were removed, and indentation was normalized for readability. Prompts show the working directory where relative filenames were used. The checker started at 11.0/25 and finished at 25.0/25.

---


## Fault 1 — Incorrect forward-zone file path

- **Symptom (what we observed):** BIND could not load the `novapeak.lab` forward zone.

  ```text
  $ python3 /root/grading/check_lab.py --group lab01 --phase build
    [   1/2 ] C4   named-checkconf -z FAILED; named-checkzone clean

  $ named-checkconf -z
  zone novapeak.lab/IN: loading from master file /etc/bind/db.MISSING failed: file not found
  zone novapeak.lab/IN: not loaded due to errors.
  _default/novapeak.lab/IN: file not found
  ```

- **Hypothesis (what we suspected and why):** The zone declaration pointed to a nonexistent file. The error named `db.MISSING`, so I checked the declaration and the files in `/etc/bind/`.

- **Proof (the command that confirmed the cause):** The declaration referenced `db.MISSING`, while the actual forward-zone file was `db.novapeak.lab`.

  ```text
  root@ubuntu:/etc/bind$ cat named.conf.local
  zone "novapeak.lab" {
      type master;
      file "/etc/bind/db.MISSING";
  };

  root@ubuntu:/etc/bind$ ls -l db.novapeak.lab
  -rw-r--r-- 1 bind bind 212 Oct 27 10:53 db.novapeak.lab

  $ ls -l /etc/bind/db.MISSING
  ls: cannot access '/etc/bind/db.MISSING': No such file or directory
  ```

- **Fix (what we changed):** Changed the `novapeak.lab` zone's file path in `/etc/bind/named.conf.local` to `/etc/bind/db.novapeak.lab`, validated with `named-checkconf -z`, and restarted `named`.

- **Confirmation (output showing it now works):** The zone loaded and C4 passed. The client still could not reach DNS, which led to Fault 2.

  ```text
  $ named-checkconf -z
  zone novapeak.lab/IN: loaded serial 2024010101

  $ python3 /root/grading/check_lab.py --group lab01 --phase build
    [   2/2 ] C4   named-checkconf -z clean; named-checkzone clean
    [   0/2 ] C1   client cannot reach named at 192.168.19.10 (timeout/no status). Check listen-on, allow-query, firewall, and that it serves on lan0.
    TOTAL: 12.0 / 25
  ```

- **Found/proved by:** Kaung Myat Tun (KingSton)

---
## Fault 2

(copy the block above for each additional fault)
--- 


## Fault 3 — Incorrect `api` A record

- **Symptom (what we observed):** Once the client could reach DNS, the checker isolated an incorrect answer for `api.novapeak.lab`.

  ```text
  $ python3 /root/grading/check_lab.py --group lab01 --phase build
    [2.25/3 ] C2   api expected 192.168.19.40 got ['192.168.19.1']
  ```

- **Hypothesis (what we suspected and why):** The DNS server was reachable and authoritative, but the `api` address was wrong. I suspected the `api` A record in the forward-zone file.

- **Proof (the command that confirmed the cause):** The zone file contained the incorrect address `192.168.19.1`.

  ```text
  root@ubuntu:/etc/bind$ cat db.novapeak.lab
  $TTL 3600
  @ IN SOA ns1.novapeak.lab. admin.novapeak.lab. ( 2024010101 3600 1800 604800 3600 )
  @ IN NS ns1.novapeak.lab.
  ns1 IN A 192.168.19.10
  www IN A 192.168.19.20
  mail IN A 192.168.19.30
  api IN A 192.168.19.1
  ```

- **Fix (what we changed):** Changed the `api` record in `/etc/bind/db.novapeak.lab` to `api IN A 192.168.19.40` and restarted `named`.

- **Confirmation (output showing it now works):** The checker then accepted all four A records. DHCP still advertised the wrong resolver, which led to Fault 4.

  ```text
  $ python3 /root/grading/check_lab.py --group lab01 --phase build
    [ 3.0/3 ] C2   all four A records correct
    [   0/3 ] C9   DHCP advertises 8.8.8.8, expected the company DNS 192.168.19.10 -- this is the integration the whole project is about
    TOTAL: 19.0 / 25
  ```

- **Found/proved by:** Lwin Oo (Edward)

---

## Fault 4 — DHCP advertised a public resolver instead of company DNS

- **Symptom (what we observed):** The client leased an address, but DHCP supplied `8.8.8.8` as its DNS server. Using that leased resolver, it could not resolve the company's `www` record.

  ```text
  $ python3 /root/grading/check_lab.py --group lab01 --phase build
    [   0/3 ] C9   DHCP advertises 8.8.8.8, expected the company DNS 192.168.19.10 -- this is the integration the whole project is about
    [   0/3 ] C11  using the leased resolver 8.8.8.8, www did not resolve to 192.168.19.20 (got nothing). If the leased resolver is a public one, that is the integration bug.
  ```

- **Hypothesis (what we suspected and why):** The company's internal names are served by `192.168.19.10`. I suspected that `/etc/dhcp/dhcpd.conf` contained the public DNS address advertised in the lease.

- **Proof (the command that confirmed the cause):** The DHCP configuration advertised `8.8.8.8`.

  ```text
  root@ubuntu:/etc/dhcp$ cat dhcpd.conf
  option domain-name "novapeak.lab";
  default-lease-time 3600;
  max-lease-time 7200;

  subnet 192.168.19.0 netmask 255.255.255.0 {
      range 192.168.19.104 192.168.19.144;
      option routers 192.168.19.1;
      option domain-name-servers 8.8.8.8;
  }
  ```

- **Fix (what we changed):** Changed `option domain-name-servers` to `192.168.19.10` in `/etc/dhcp/dhcpd.conf`. Initially I ran `systemctl restart named`, which restarted DNS rather than DHCP; the next checker run still reported `8.8.8.8` and a score of 19.0/25. From `/etc/dhcp/`, I then ran `dhcpd -t -cf dhcpd.conf`, which reported no configuration errors, followed by `systemctl restart isc-dhcp-server`. The validation checked the file; restarting the DHCP service applied the change.

- **Confirmation (output showing it now works):** The client then received the company DNS server and resolved `www` through its leased resolver. The full checker result was 25.0/25.

  ```text
  $ python3 /root/grading/check_lab.py --group lab01 --phase build
    [   2/2 ] C4   named-checkconf -z clean; named-checkzone clean
    [   2/2 ] C5   dhcpd -t clean; service running
    [   2/2 ] C1   named is authoritative for novapeak.lab and reachable from the client.
    [ 3.0/3 ] C2   all four A records correct
    [   2/2 ] C3   reverse lookup of 192.168.19.10 returns ns1.
    [   3/3 ] C6   client obtained a lease: 192.168.19.104
    [ 2.0/2 ] C7   pool+netmask correct
    [   2/2 ] C8   gateway advertised correctly
    [   3/3 ] C9   DHCP advertises the company DNS (192.168.19.10)
    [   1/1 ] C10  company domain advertised
    [   3/3 ] C11  client resolves www via the leased resolver
    --------------------------------------------------
    TOTAL: 25.0 / 25
  ```

- **Found/proved by:** Lwin Oo (Edward)