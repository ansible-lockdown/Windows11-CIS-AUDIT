# Windows 11 Enterprise Audit

## Overview

### Based on CIS Microsoft Windows 11 Enterprise Benchmark v3.0.0

[Centre For Internet Security]

This repository is a set of configuration files and directories to run the audit
of the relevant benchmark of Windows 11 Enterprise workstations.

It is configured in a directory structure level, one file per control, matching
the layout of the Linux audit repositories.

## What this audit asserts, and what it deliberately does not

A Windows setting and the registry key a playbook writes are not always the same
thing. Account policy and account lockout policy live in the SAM and are only
reachable through `secedit`; a registry write to `Control\Lsa` reports success
and applies nothing. So sections 1.1, 1.2 and the `System Access` controls in
2.3.x are audited through the effective policy, never through the key the
remediation role happens to write.

Those controls are expected to fail against a role that still writes the
registry. That is the point of the audit, and softening it would produce a green
result that proves nothing.

## Which binary runs this

The content is Goss format, the same as every other Ansible-Lockdown audit
repository. It is run with [syver](https://github.com/krameff/syver), the Goss
fork that adds Windows support.

The fork matters here rather than being a preference. Of the 540 specs, **403
assert through the `registry:` resource**, 137 through `file:` and 6 through
`command:`. `registry:` is provided by syver, so upstream Goss can run the
`file:` and `command:` subset but not the three quarters of this benchmark that
reads the registry. Anything that implements the same resources will run the
content unchanged - nothing here is syver-specific beyond the resource types.

## Requirements

- `syver` >= 0.11.0. Published releases are at
  <https://github.com/krameff/syver/releases>; `v0.11.1` is the version this
  content is validated against.

  From 0.11.0 syver reports a check it cannot run as an error rather than
  letting it pass quietly. Below that version a green result is worth slightly
  less, because an unsupported assertion can look like a pass. `run_audit.ps1`
  warns rather than refusing to run, and everything this audit uses -
  `registry:`, `file: contents:`, `command:`, `--vars`, `--use-alpha=1` - works
  as far back as 0.9.4.

  Nothing in this repository downloads the binary. Run standalone, pre-stage it
  and set `AUDIT_BIN` if it is not at `C:\Program Files\syver\syver.exe`. Run
  from the remediation role, `get_audit_binary_method: download` fetches the
  published asset and verifies its SHA256.
- Administrator privileges. `secedit`, `auditpol` and `HKEY_USERS` all need them.
- Windows support in syver is alpha and gated behind `--use-alpha=1`, which
  `run_audit.ps1` passes for you.

A Defender exclusion for the syver binary is worth adding on audit hosts. An
unsigned binary is scanned on every launch, and that is most of the cold-start
cost.

## The policy snapshot

Most of this benchmark is not readable by a goss resource. `run_audit.ps1`
therefore collects once, up front - one `secedit /export`, one `auditpol`, one
`Get-Service`, one `Get-NetFirewallProfile`, one pass over the loaded user hives
- normalises the result, and writes a snapshot to
`%TEMP%\win11cis_policy_snapshot.txt`. The controls match against that with
plain `file:` checks.

**The audit writes that file.** It is a policy export in the temp directory. The
alternative was roughly 130 concurrent `secedit` exports contending on LSA.

What gets collected comes from `collector_targets.json`, which the generator
writes alongside the specs, so the set of things collected and the set of things
asserted cannot drift apart.

```powershell
.\run_audit.ps1 -CollectOnly      # collect and stop, to read the snapshot
.\run_audit.ps1 -SkipCollect      # reuse an existing snapshot. It may be stale.
```

Each control reads one small file per collected value, under
`%TEMP%\win11cis_snapshot\`, rather than the combined snapshot. goss prints the
whole file when a `contents:` matcher fails, so a shared snapshot would bury
every failure under every other value and make the report unreadable.

Two checks in `goss.yml` assert that the snapshot exists and is complete, so a
missing or half-written snapshot fails loudly instead of reading as a clean run.

An empty value is recorded as the literal `NONE`, never as nothing. goss splits
a file into lines and an empty file has none, so a `/^$/` matcher can never
match: every "No One" user-right control would fail while the host was in fact
compliant.

## Variables

file: `vars/CIS.yml`

Section toggles, the three profile toggles (`win11cis_level_1`,
`win11cis_level_2`, `win11cis_bitlocker`) and one toggle per control. Site
values - password lengths, log file sizes, the logon banner, which principals
should hold a configurable user right - sit at the end of the file.

Two of them mirror gates in the remediation role and must match how it was run:

- `win11cis_domain_joined` - several controls apply only to domain members, and
  four BitLocker controls only to standalone machines.
- `win11cis_win_skip_for_test` - the role's own switch for the 13 controls that
  would sever a test host (RDP, WinRM, sshd, the public firewall). If the role
  was run with it set, set it here too, or those controls will fail correctly
  and unhelpfully.

## Section 19 needs a loaded user hive

Section 19 audits per-user policy under `HKEY_USERS`. Only hives for logged-on
users are loaded. The remediation role's prelim `REG LOAD`s every profile; an
audit must not, because that changes the host.

So the collector examines the hives that are loaded and records how many. On a
freshly booted machine with no interactive session there are none, and all 13
controls fail with `no_user_hives_loaded`. That is a precondition of a
meaningful section 19 result, not a defect.

## Coverage

<!-- BEGIN COVERAGE (generated by scripts/generate_windows_audit.py - do not edit) -->

| Section | In benchmark | Asserted | Coverage |
| --- | --- | --- | --- |
| 1 | 11 | 11 | 100% |
| 2 | 105 | 105 | 100% |
| 5 | 44 | 44 | 100% |
| 9 | 23 | 23 | 100% |
| 17 | 27 | 27 | 100% |
| 18 | 317 | 317 | 100% |
| 19 | 13 | 13 | 100% |
| **All** | **540** | **540** | **100.0%** |

### Controls that branch on state discovered at run time

These pick *which* value is correct from something a gossfile
template cannot see - whether Hyper-V or IIS is installed, whether
an account has been renamed, what a site variable was set to. Each
asserts the part that holds on every host; the feature-dependent
part is matched optionally rather than required.

2.2.14, 2.2.24, 2.2.29, 2.3.1.4, 2.3.1.5, 18.10.86.2, 18.10.92.2.1, 18.10.92.2.2

<!-- END COVERAGE -->

## Usage

```powershell
# stage the content and the binary, then
.\run_audit.ps1 -Format documentation
.\run_audit.ps1 -Format json -OutFile C:\audit\win11.json
```

`-Format documentation` shows every assertion and its outcome. The default
output is terser and easier to over-read.

For the latest information on audit and how it can be used please visit

[Read the Docs - Audit]

## Branches

If running as part of the Ansible playbook, this will pull in the relevant
branch for the version of benchmark you are remediating.

- e.g. v3.0.0 will pull in branch benchmark-v3.0.0

Devel is normally the latest benchmark version, so may differ from the version
of benchmark you wish to test.

## Support

[Discord Community Discussions]

[Enterprise Support]

[MindPoint Group]

## Contributing

Bug reports and feature requests are welcome from everyone, please raise an issue.

Pull requests are accepted from approved contributors only. To be onboarded, join the [Discord Server](https://www.lockdownenterprise.com/discord) and request contributor access. See [CONTRIBUTING.md](CONTRIBUTING.md) for the full process.

## Links and Further information

- [Centre For Internet Security]

<!----
README Links
---->

[benchmark-type]: CIS
[OS-VERSION]: Windows11
[os-type]: Windows
[Centre For Internet Security]: https://www.cisecurity.org
[Read the Docs - Audit]: https://ansible-lockdown.readthedocs.io/en/latest/audit/getting-started-audit.html
[MindPoint Group]: https://mindpointgroup.com/cybersecurity-consulting/automate/baseline-modernization#GH_LockdownReadMe
[Discord Community Discussions]: https://www.lockdownenterprise.com/discord
[Enterprise Support]: https://lockdownenterprise.com#GH_LockdownReadMe
