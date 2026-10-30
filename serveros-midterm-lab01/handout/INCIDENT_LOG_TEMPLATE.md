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
