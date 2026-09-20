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
| pentestvm | ubuntu-noble-x86_64        | 192.168.56.10   | Trainee workstation. |
| web01     | ubuntu-noble-x86_64        | 192.168.56.58   | Apache on 80/443. Self-signed cert (`CN=web01.cadmus.local`), status page, robots.txt. |
| dev01     | ubuntu-noble-x86_64        | 192.168.56.75   | vsftpd with anonymous read, `git daemon` on 9418 serving `web-inventory.git`, dev portal on 8080. |
| mail01    | ubuntu-noble-x86_64        | 192.168.56.127  | Postfix on 25. |
| db01      | ubuntu-noble-x86_64        | 192.168.56.152  | PostgreSQL on 5432. |
| mon01     | ubuntu-noble-x86_64        | 192.168.56.186  | snmpd with a read-only community, metrics endpoint on 9100, dashboard on 3000. |
| file01    | windows-server-2019-x86_64 | 192.168.56.201  | SMB, RDP and WinRM. Shares require credentials. |
| ws01      | windows-10-x86_64          | 192.168.56.247  | SMB. The `Shared` share is listable and readable anonymously. RDP is switched off. |

`file01` and `ws01` run on `c2_r4_d40`; everything else is `standard.small`.

## Provisioning

`provisioning/playbook.yml` runs a common baseline across the Linux hosts and
then one play per target:

- **baseline** — disables unattended upgrades, installs common packages, sets
  hostnames, writes an SSH login banner, and seeds `/etc/hosts` on the targets
  only. `pentestvm` is deliberately excluded from that last task: seeding it
  there would hand the trainee the inventory the lab asks them to build.
- **pentestvm** — installs the scanning toolset and provisions the trainee
  login `user` / `Password123` (sudo) via the `user-access` role.
- **web01 / dev01 / mail01 / db01 / mon01** — deploy the services above.
- **file01 / ws01** — set hostnames, configure SMB, and on `ws01` relax the LSA
  registry settings so a null session can list shares.

Windows hosts are configured over WinRM and need the `ansible.windows`
collection, declared in `provisioning/requirements.yml`.

## Trainee workflow

1. Console into **pentestvm** as `user` / `Password123`.
2. Establish position on the network (`ip addr`, `ip route`).
3. Sweep `192.168.56.0/24` for live hosts.
4. Fingerprint each one — the default port set is not enough; several services
   sit on ports a default scan never checks.
5. List shares on the SMB hosts. One answers without credentials.
6. Submit the name of that share.
7. A ten-question knowledge check follows, drawing on findings from across the
   whole range.

## Flags

`variables.yml` is empty: this lab uses static answers rather than APG
variables, because what is being graded is information discovered by scanning
rather than a planted secret. Two `cadmus{...}` strings are planted as
assessment answers, on dev01's anonymous FTP root and mon01's metrics endpoint.
A third sits in a web01 response header and is currently unused.

## Tools used

`nmap` (including NSE), `arp-scan`, `smbclient`, `snmpwalk`, `ftp`, `curl`,
`openssl`, `netcat`, `psql`.

## MITRE mapping

- `T1046` Network Service Discovery
- `T1018` Remote System Discovery
- `T1016` System Network Configuration Discovery
- `T1135` Network Share Discovery
