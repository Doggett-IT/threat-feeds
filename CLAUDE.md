# threat-feeds

IP blocklist feeds used to flag or block SSL VPN logins originating from
hosting, datacenter, VPN, and proxy infrastructure rather than from
legitimate user networks.

## Files

| File | Role |
| --- | --- |
| `hosting-ips-consolidated.txt` | The curated feed. Hand-maintained, strictly sorted. **New entries go here.** |
| `hosting-ips` | A much larger raw feed. Not curated; not the file to edit when adding an address. |

## Adding an address

1. **Look it up in RDAP.** `rdap.org` returns 403 to automated requests. Query
   `https://rdap.arin.net/registry/ip/<ip>` directly — it answers with a 303
   redirect to the correct regional registry (RIPE, APNIC, etc.) when the
   address falls outside ARIN's region.

2. **Confirm the registrant is actually hosting infrastructure.** Check
   unfamiliar organization names against their own website before deciding.
   Names are often uninformative: "Ethernet Servers Ltd" is a UK VPS and
   dedicated-server host; IPXO is an address-leasing marketplace whose blocks
   are commonly sub-leased to hosting and proxy operators.

3. **Apply the residential ISP exception.** See below. This is the rule with
   real blast radius.

4. **Size the CIDR from RDAP's most specific record** (the SWIP reassignment,
   where one exists). If the file already holds an entry for the same
   organization at a narrower size than the parent allocation, match that
   existing granularity rather than widening the block.

   Example: `163.245.192.0/24` was added next to an existing
   `163.245.222.0/24` from the same ARIN `/19`, rather than blocking the whole
   `/19`.

5. **Insert in ascending numeric order per octet.** This is *not* lexicographic
   order — `24.167.x` sorts before `31.222.x`, and `104.238.x` after
   `104.223.x`. Check the surrounding lines before writing.

6. **Write the reasoning into the commit message.** Name each range, its
   registrant, and why it was classified the way it was, so the next edit can
   be understood without repeating the lookups.

## Do not block residential ISP subnets

When RDAP returns a legitimate consumer ISP or telecom — Charter, Windstream,
a regional fiber carrier — add the **single offending address as a `/32`** and
nothing wider.

This rule exists because it was broken once. Commit `76e8b85` reverted a
Windstream `/10` and a Missouri Network Alliance `/12` that had been added
wholesale; either would have locked out every legitimate remote user behind
those carriers.

## Commit message convention

Summary line names what changed, body explains per-range. For example:

```
Add 5 lockout-hit IPs to hosting/VPN feed after RDAP triage

- 45.135.160.0/22 - IPXO LLC, an IP address leasing marketplace whose
  blocks are commonly sub-leased to hosting/proxy/VPN operators.
...

24.167.0.153 (Charter Communications/Road Runner) is legitimate US
residential ISP space, so per the feed's classification rule it was
added as a /32 only, not a subnet block.
```

## Pushing from Windows (machine-specific)

If `git push` over SSH fails with `Permission denied (publickey)` on a Windows
workstation, run `ssh -v` first. **If the output contains
`debug1: Server accepts key:`, the key is registered and authorized** — the
remote side is satisfied and the problem is entirely local. Re-adding keys on
github.com will not help.

The usual local cause is that the `ssh` bundled with Git for Windows
(`/usr/bin/ssh` under Git Bash) cannot reach the Windows ssh-agent's named
pipe, so it never finds the unlocked key. Point git at the native client
instead:

```
git config core.sshCommand '"C:/Windows/System32/OpenSSH/ssh.exe"'
```

Note this is per-repository, and that Git Bash may resolve `$HOME` to
something other than `%USERPROFILE%` — an empty `~/.ssh` is not evidence that
no keys exist.

---

*Provenance: the classification rules above were reconstructed from
`git log -p` on `hosting-ips-consolidated.txt` in September 2026, not from a
written policy. They match every precedent in the history, but correct this
file if the intended policy differs.*
