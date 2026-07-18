# Incident Report: EC2 Instance Ran Out of Disk Space

## What happened

The `storage-breaker` app was deployed on a small EC2 instance (t2.micro, 10 GiB root volume). The app exposes a `/health` endpoint that normally returns `{"status":"healthy"}`. After running for a while, it started returning `{"status":"unhealthy"}` with an HTTP 503 instead.

The cause turned out to be disk space. The app was writing continuously to a single log file at `/var/log/storage-breaker/application.log`, and nothing was in place to rotate or clean it up. The file kept growing until it consumed the entire 10 GiB disk. Once the disk was full, the app could no longer operate normally and its internal health check flagged it as unhealthy.

## Timeline (UTC)

- **15:43** : App started via systemd. `/health` returns `200 healthy`.
- **16:57** : Disk usage checked: already at 79%. Log folder alone measured 5.3 GB.
- **17:22** : `/health` now returns `503 unhealthy` — first observed failure.
- **17:30** : Disk fully exhausted: 100% used, 0 bytes free. Even `mkdir` failed with "No space left on device." Log folder had grown to 7.3 GB.
- **17:33** : Log file cleared with `truncate` to restore service. Disk usage dropped to 28%, and `/health` returned to `200 healthy` immediately.
- **17:36–17:40** : Log rotation configured and verified. Disk usage stabilized around 25%.

From a healthy start to full disk exhaustion took under two hours driven entirely by uncontrolled log growth.

## Root cause

The app logged everything to one file with no rotation, retention, or size limit in place. On a larger volume this might not have mattered for days or weeks, but with only 10 GiB available, the disk filled quickly enough to cause an outage in under two hours.

## Detection

The issue was caught through manual checks by polling `/health` with `curl` and checking disk usage with `df -h` at intervals. No automated alerting was configured for this drill.

One notable finding: **EC2's built-in status checks did not detect this at all.** Those checks only verify that the instance is reachable and the OS is responsive that they have no visibility into disk usage. Even while the app was actively returning `503 unhealthy`, the EC2 console's status checks would most likely have continued showing the instance as healthy. This highlights a real gap between "the infrastructure is up" and "the application is actually functioning" : infrastructure-level checks and application-level health checks are not the same thing, and relying on only one can miss real problems.

## Resolution

**Immediate mitigation (restore service):**
```bash
sudo truncate -s 0 /var/log/storage-breaker/*
```
This freed disk space instantly without requiring a restart. `/health` returned to `200 healthy` right after.

**Permanent fix (prevent recurrence):**
A `logrotate` configuration was added at `/etc/logrotate.d/storage-breaker`:
```
/var/log/storage-breaker/*.log {
    size 100M
    rotate 3
    compress
    missingok
    notifempty
    copytruncate
}
```
- `size 100M` : rotates once the log hits 100 MB, rather than waiting on a fixed schedule
- `rotate 3` : keeps only the 3 most recent rotated copies
- `compress` : gzips old logs to save space
- `copytruncate` : rotates in place without needing to restart the running Uvicorn process

One thing worth calling out: `logrotate` runs once a day by default via cron. That's far too infrequent here that the app can generate 100+ MB of logs within minutes, so a daily check would never catch it in time. A dedicated cron entry was added instead to run logrotate every 5 minutes:
```
*/5 * * * * /usr/sbin/logrotate /etc/logrotate.d/storage-breaker
```

**Verification:** after roughly 10 minutes of operation under the new configuration, disk usage held steady around 24–25%, with rotated and compressed log files (`application.log.1.gz`, `.2.gz`, `.3.gz`) confirming rotation was actively running on schedule.

## Lessons learned

- Application-level health checks can catch failures that infrastructure-level checks completely miss. EC2 status checks and default CloudWatch metrics wouldn't have flagged this incident at all.
- Log rotation intervals need to match the actual write rate of the application. A default daily schedule is not universal and can be dangerously slow for high-volume logging.
- Small root volumes (10 GiB) leave very little margin for unbounded log growth; either a larger/dedicated log volume or more aggressive rotation is needed for anything log-heavy.
- Manual polling worked for this drill, but a production setup would benefit from the CloudWatch Agent tracking `disk_used_percent` with a proper alarm, instead of relying on someone checking manually.

## Supporting evidence

```
# At the point of failure:
$ df -h /
/dev/root       9.6G  9.5G     0 100% /

$ du -sh /var/log/storage-breaker
7.3G    /var/log/storage-breaker

$ curl -i http://127.0.0.1/health
HTTP/1.1 503 Service Unavailable
{"status":"unhealthy"}

# Immediately after mitigation:
$ curl -i http://127.0.0.1/health
HTTP/1.1 200 OK
{"status":"healthy"}

# ~10 minutes after log rotation was configured:
$ ls -la /var/log/storage-breaker
-rw-rw-r-- 1 ubuntu ubuntu 53084160 Jul 18 17:40 application.log
-rw-rw-r-- 1 ubuntu ubuntu   725435 Jul 18 17:40 application.log.1.gz
-rw-rw-r-- 1 ubuntu ubuntu    77438 Jul 18 17:37 application.log.2.gz
-rw-rw-r-- 1 ubuntu ubuntu  2192966 Jul 18 17:36 application.log.3.gz
```
