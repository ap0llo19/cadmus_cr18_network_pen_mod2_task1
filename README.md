# CR-18 / Module 2 / N2c — Internal Network Enumeration

CADMUS Cyber Range scenario. An internal asset assessment with no documentation
to work from: the trainee sweeps a flat `/24`, fingerprints every service it
turns up, infers each host's role, and finishes by naming a workstation share
that is readable with no credentials at all.

## Topology

A single `192.168.56.0/24` segment behind one router, with a `10.10.10.0/24` WAN.
Addresses are deliberately non-sequential so the range has to be swept rather
than guessed.

| Node      | Image                      | IP              | Role |
| --------- | -------------------------- | --------------- | ---- |
| router    | debian-12-x86_64           | 192.168.56.1    | LAN gateway. Not a target. |
| vma       | kali-2026.1-x86_64         | 192.168.56.10   | Trainee workstation. |
| web01     | ubuntu-noble-x86_64        | 192.168.56.58   | Apache on 80/443. Self-signed cert (`CN=web01.cadmus.local`), status page, robots.txt. |
| dev01     | ubuntu-noble-x86_64        | 192.168.56.75   | vsftpd with anonymous read, `git daemon` on 9418 serving `web-inventory.git`, dev portal on 8080. |
| mail01    | ubuntu-noble-x86_64        | 192.168.56.127  | Postfix on 25. |
| db01      | ubuntu-noble-x86_64        | 192.168.56.152  | PostgreSQL on 5432. |
| mon01     | ubuntu-noble-x86_64        | 192.168.56.186  | snmpd with a read-only community, metrics endpoint on 9100, dashboard on 3000. |
| file01    | windows-server-2019-x86_64 | 192.168.56.201  | SMB, RDP and WinRM. Shares require credentials. |
| ws01      | windows-10-x86_64          | 192.168.56.247  | SMB. The `Shared` share is listable and readable anonymously. RDP is switched off. |

`vma` runs on `c2_r4_d30` (Kali's image sets a 25 GiB `min_disk`, so `standard.small` is not an option). `file01` and `ws01` run on `c2_r8_d40`; everything else is `standard.small`.

## Provisioning

`provisioning/playbook.yml` runs a common baseline across the Linux hosts and
then one play per target:

- **baseline** — disables unattended upgrades, installs common packages, sets
  hostnames, writes an SSH login banner, and seeds `/etc/hosts` on the targets
  only. `vma` is deliberately excluded from that last task: seeding it
  there would hand the trainee the inventory the lab asks them to build.
- **vma** — installs the scanning toolset and provisions the trainee
  login `user` / `Password123` (sudo) via the `user-access` role.
- **web01 / dev01 / mail01 / db01 / mon01** — deploy the services above.
- **file01 / ws01** — set hostnames, configure SMB, and on `ws01` relax the LSA
  registry settings so a null session can list shares.

Windows hosts are configured over WinRM and need the `ansible.windows`
collection, declared in `provisioning/requirements.yml`.

## Trainee workflow

Eleven levels, 200 points, roughly 50 minutes. Five graded hands-on levels,
then a short knowledge check.

1. Open the Kali desktop on **vma** (`Open GUI`) and log in as `user` / `Password123`.
2. *Background* — locating yourself and sweeping.
3. **Map the segment** — pick the lab interface out of the two present, sweep
   it, submit the number of addresses that reply (10: seven targets, the
   workstation, the gateway, and one the platform itself holds).
4. *Background* — interrogating services, including the NSE scripts `-sC`
   does not run.
5. **Login without a password** — anonymous FTP on dev01.
6. **Services that ask nothing** — the unauthenticated metrics endpoint on
   mon01:9100. SNMP on the same host is the same class of finding.
7. **Ask the right question** — `ssh2-enum-algos` against db01; submit the
   first cipher the server offers.
8. **File access with no login** — null-session SMB share listing on ws01.
9. **Knowledge check** — five questions on web01, mail01, file01 and dev01,
   all answerable from scan output already produced.

The four middle levels are deliberately one theme: four different ways into a
system that ask for no credentials at all.

## Flags

`ftp_flag` and `metrics_flag` are APG variables, generated per sandbox and
templated into dev01's FTP readme and mon01's metrics endpoint, so answers
cannot be shared between trainees. They are raw generated values — APG cannot
produce a `cadmus{...}` wrapper, which is why Module 3 dropped it too.

The share name in the final level is fixed at `Shared`: randomising it would
cost the realism that makes the finding recognisable. The remaining
`ASSESSMENT_LEVEL` answers are static by necessity — assessment questions
cannot bind APG variables.

## Tools used

`nmap` (including NSE), `arp-scan`, `smbclient`, `snmpwalk`, `ftp`, `curl`,
`openssl`, `netcat`, `psql`.

## MITRE mapping

- `T1046` Network Service Discovery
- `T1018` Remote System Discovery
- `T1016` System Network Configuration Discovery
- `T1135` Network Share Discovery
