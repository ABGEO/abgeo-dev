+++
title = "php2shell, Two Critical PHP Bugs Leaked Before the Patch"
date = "2026-10-21T12:13:26+04:00"
cover = "/blog/php2shell-critical-php-rce/images/cover.png"
images = [
    "/blog/php2shell-critical-php-rce/images/cover.png"
]
tags = ["Security", "PHP", "Vulnerability Management", "DevOps"]
keywords = [
    "php2shell",
    "cve-2026-109618",
    "cve-2026-109619",
    "php rce vulnerability",
    "php critical security release october 2026",
    "php core remote code execution",
    "php 8.5.12",
    "php end of life security",
    "php 7 no security fix",
    "php-fpm exploit detection",
    "php vulnerability mitigation",
    "php embargo leak"
]
description = "Two critical PHP core bugs leaked a day before the embargo. The fix ships on October 21, and most PHP servers on the internet will not get it."
showFullContent = false
readingTime = true
+++

PHP ships an emergency security release today, October 21, 2026, at 14:00 UTC. The release fixes two critical bugs in
PHP core. People already call them "php2shell". The details leaked one day before the embargo ended, so the clock
started early for everybody who runs PHP.

I wrote this post with what was public at 08:00 UTC on October 21. The fix is out now, so start with the next section.

## Update: the fix is out

PHP released 8.5.12, 8.4.27, 8.3.36 and 8.2.35 on October 21. Read the
[security advisory](https://www.php.net/archive/2026.php#2026-10-21-5) and update now. The advisories list every
version below these releases as affected. Both bugs work without authentication.

**The bugs.**

- CVE-2026-109618, "RCE via OOM corruption"
  ([GHSA-w8hw-w9fx-j79w](https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w)). An out-of-memory
  error can stop PHP while request data is only partly initialized. Request cleanup then uses the broken data. The
  trigger is a crafted multipart request.
- CVE-2026-109619, "RCE via stale global state"
  ([GHSA-6wmf-4rv5-p3rp](https://github.com/php/php-src/security/advisories/GHSA-6wmf-4rv5-p3rp)). A fatal error can
  leave a worker with pointers into memory of a finished request. The next request in the same worker can use them
  before PHP resets them. Session upload progress, which is on by default, is one of the confirmed paths. So the
  write-up that pointed at the upload progress handler was right, at least for this bug.

**Who is at risk.** All PHP installations that serve web requests: PHP-FPM, mod_php, php-cgi, FrankenPHP and LiteSpeed.
The risk is highest for mod_php. There is a working exploit where an unauthenticated request triggers the attack before
any application code runs. For PHP-FPM, the advisory has no complete exploit yet. Do not take that as a reason to wait.

**Old versions.** PHP 8.1 and older still get no official release. The PHP security team published minimal source
backports for [PHP 7.4](https://github.com/php/php-src/tree/PHP-7.4-CVE-2026-109618-109619),
[PHP 8.0](https://github.com/php/php-src/tree/PHP-8.0-CVE-2026-109618-109619) and
[PHP 8.1](https://github.com/php/php-src/tree/PHP-8.1-CVE-2026-109618-109619). They help if you build PHP from source.
If you use packages, ask your distribution or LTS vendor. There is nothing for PHP 7.3 and older. Note that these
backports turn off session upload progress completely, so that feature stops working after you apply them.

**If you cannot update today.** The advisory lists settings that reduce the risk. None of them makes a server safe.

- Set `session.upload_progress.enabled=0` in `php.ini` or in the FPM pool configuration, then restart every worker.
  `ini_set()` and `.user.ini` do not help here. If you do not use this feature, keep it off after the update too.
- On PHP 8.5, set `fatal_error_backtraces=0` and restart every worker.
- Set a low `post_max_size`: 8M if the application takes no file uploads, 32M if it does. Also limit request size at
  the web server or proxy.
- Do not use `RLIMIT_AS` or `RLIMIT_DATA` to limit memory. They can trigger the OOM bug.

The advisory also mentions `memory_limit=-1` for the OOM bug. Read its notes before you use it, because it removes a
limit that many servers rely on.

**Update (October 21):** The PHP Foundation
[warns](https://x.com/ThePHPF/status/2112828424486056183) that the volunteer-run Remi repository
(rpms.remirepo.net) will not have updates today. If you install PHP from Remi, switch to your distribution's packages or
build from source, so you can apply the fix as soon as it is out.

**Update (October 21):** Cloudflare published an
[emergency WAF release](https://developers.cloudflare.com/changelog/post/2026-10-21-emergency-waf-release-cfmr/)
with four new "Anomalous Execution" rules in its Managed Ruleset. The changelog does not name these CVEs, but the
timing suggests a link. If your site is behind Cloudflare, check that the Managed Ruleset is enabled and that your
traffic really goes through Cloudflare. Do not treat this as a fix. A WAF rule can miss a variant of the attack, and
requests that reach your server directly skip it. You still need to update PHP.

**Update (October 21):** Some organizations are taking websites offline until the fix is out. Finland's state IT
centre Valtori
[took public government websites offline](https://www.devdiscourse.com/article/international/3995774-finland-takes-public-government-websites-offline-as-precaution-over-php-concern)
and says this is a precaution, not the result of an attack or a breach. In the Netherlands, the
[NOS reports](https://nos.nl/liveblog/2636066-websites-kwetsbaar-voor-softwarelek-rtl-nieuws-digid-en-mijnoverheid-uit-de-lucht)
that the parliament website, DigiD.nl, MijnOverheid.nl and several hospital sites are offline. If you cannot patch
quickly, a few hours offline is a valid choice too (see step 8 below).

## What happened

On October 20 at 15:20 UTC, a "Tell HN" post on Hacker News said that two PHP remote code execution (RCE) disclosures
were near. The author referred to a "reliable source" and gave no evidence. The post was flagged soon after. That did
not help much, because screenshots were already on X and Mastodon.

Then [watchTowr](https://labs.watchtowr.com/) confirmed that the bugs are real. They also said that they reproduced
CVE-2026-109618 in their lab. The embargo was to end on Thursday, October 22.

Some hours later, the PHP security team put a notice on
[php.net](https://www.php.net/archive/2026.php#2026-10-20-1). They moved the release one day earlier. They also kept the
right to release sooner if the details become public or attacks start.

I do not like how this started. An embargo gives defenders time to prepare. A rumor post without evidence took one day
of that time from every team that runs PHP. The PHP team made a good decision here: they moved the date forward and did
not wait for the leak to get worse.

## What is confirmed

The php.net notice gives very little. There are two critical vulnerabilities, and each one has a CVSS v4.0 score of
9.2. The notice does not give CVE IDs. It also does not give affected versions or a bug description. PHP will publish
these with the release.

The CVE IDs (CVE-2026-109618 and CVE-2026-109619) and the "RCE in PHP core" description come from researchers, not
from PHP. The leaked post said CVSS 10.0. That number is not correct. The official score is 9.2.

## What is not confirmed

I wrote this section before the release. The update at the top answers most of these points.

**Root cause.** Nobody has published it. One third-party write-up points at the session upload progress handler. No
vendor confirms this, so I do not recommend that you build your plan on it.

**Preconditions.** We do not know if the attacker must be authenticated. We also do not know if a special configuration
is necessary. A 9.2 score suggests that a plain HTTP request can be enough, but that is a guess.

**Affected versions.** [Sansec](https://sansec.io/research/php2shell) reports that all releases from PHP 5.4 are
vulnerable. PHP did not confirm this.

**Fixed versions.** Trackers expect 8.5.12, 8.4.27, 8.3.36 and 8.2.35. These numbers follow the September 24 releases.
They are not official until php.net publishes them.

## Who gets a fix

PHP supports only the 8.2 branch and later. Security support for 8.2 stops on December 31, 2026. If your
server runs PHP 8.1 or older, you will not get an official fix. Since the release, there are source backports for
PHP 7.4, 8.0 and 8.1 (see the update at the top).

This is the part that worries me. [Mallory](https://mallory.ai/blog/php-rce-rapid-response) counted internet-facing
hosts on Shodan on October 20. About 1.5 million hosts show a PHP version. Of these, 883,717 (59%) run PHP 8.1 or older.
Another 740,000 hosts hide their version, so the real number is higher. PHP 5.6 and 7.2 together run on about 371,000
of them. PHP 5.6 lost support at the end of 2018. I am not surprised, but I am a bit sad.

Some Linux distributions backport security fixes to old PHP packages. Debian and Ubuntu often do this for their own
supported releases. Check your distribution's security tracker. Do not assume that your version number tells the full
story.

## What to do before 14:00 UTC

1. Find all PHP installations that you are responsible for.
2. Include the hidden ones. Look in containers and vendor appliances. On Windows, look for software that includes
   `php.exe` or `php-cgi.exe`.
3. Record the PHP version and the SAPI (for example PHP-FPM or mod_php) for each host.
4. Put internet-facing hosts first in your patch queue.
5. If a hosting provider controls your PHP build, contact them now. Ask when they will apply the fix.
6. If you install PHP from the Remi repository on RHEL, Rocky, AlmaLinux or Fedora, plan another source for today. The
   PHP Foundation says that Remi will not have the update today. Use your distribution's packages or build from source.
7. For PHP 8.1 or older, start the upgrade to a supported branch.
8. If you cannot upgrade an old host today, remove it from the public internet. A VPN or an IP allowlist is enough for
   now.

Sansec makes an important point for online stores on Magento and similar platforms. The bug is in PHP itself. The attack
runs before your application code. A plugin or an application-level patch cannot fix it. You must update PHP on the
server.

## What to do after the release

1. Read the official release notes. Compare the affected versions with your inventory.
2. Install the new PHP packages.
3. Restart PHP-FPM or the web server. Use a full restart, not a reload.
4. Make sure that the web server runs the new version. Do not trust `php -v` alone. The CLI and PHP-FPM can be
   different binaries.

Do not skip step 3. A new package on disk does nothing while the old worker processes still run. A server can show
"patched" in the package manager and still serve requests with the old binary.

Why a full restart? The patch also changes OPcache. Before the fix, a memory error at the wrong moment could leave bad
data in the OPcache shared memory, which all workers use. A full restart throws that memory away. A reload or
`opcache_reset()` may also do it, but I did not verify that, so I would not rely on it today.

Also watch your logs after the update. The patch changes how PHP handles memory errors. In some cases a worker now exits
instead of showing the usual "Allowed memory size exhausted" fatal error. On 8.5.12 this can happen when a request goes
far over `memory_limit` (see the new `memory_hard_limit_percent` setting). With the backports for PHP 7.4 to 8.1, it
can already happen at `memory_limit`. The message then goes to stderr, not to your PHP error log, and the web server or
PHP-FPM logs that a child process exited. This does not mean that the update broke your server. It means that a request
used too much memory.

Sansec expects mass attacks within 15 minutes of the release. That sounds dramatic. For a bug in PHP core with a public
diff, I think it is realistic.

## What to look for

On October 20, Huntress and watchTowr reported no exploitation in the wild. There are no confirmed indicators of
compromise yet. [Huntress](https://www.huntress.com/blog/php2shell-leaked-critical-php-vulnerabilities) gives a good
behavior signal: a PHP worker process that starts an unexpected child process.

Monitor these parent processes:

```
php-fpm, php-cgi, php-cgi.exe, httpd, apache2
```

Alert when they start one of these:

```
sh, bash, cmd.exe, powershell.exe, python, perl, curl, wget, nc, certutil.exe
```

Also look for new files in web roots. New cron jobs and new user accounts are also signs of a problem. Normal PHP
applications almost never start `bash` or `curl` as a child process. That makes this alert useful even after the patch.

One more warning. One write-up mentions a "PoC" archive for this bug on a third-party host. Do not run it on your
workstation. A fake
exploit for a hot bug is an easy way to put malware on the machines of security people.

## My take

If you run PHP 8.2 or later, today is a normal patch day with a hard deadline. If you have a PHP 7.4 server on a VPS
that nobody remembers, today is the day to find it. Somebody else will look for it tonight.
