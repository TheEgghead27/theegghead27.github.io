# Postmortem on the Stuyvesant Student Union's CapRover crash (2022-08-31 - 2022-09-01)

## Summary
Between the hours of 20:00 and 00:20 (EDT) on August 31 and September 1, 2022, developers encountered NetworkErrors with Google OAuth.
Upon further investigation, the CapRover instance had crashed too, and some sites and services became inaccessible.

The event was triggered by a full disk on the DigitalOcean Droplet used to host the Student Union infrastructure, which led to unexpected crashes.

The team addressed the event by logging into the server and cleaning out unused cache data to free up over 50% of the server capacity.

This major incident did not seem to affect many users, as it was during off-peak hours and before the school term began.

## Primary Incident
### Leadup
Between 08/25 12:00 to 08/26 0:00, these configurations the server's disk usage hit 100%, potentially leading to the saturation of our HoneyBadger reporting capacity.

### Fault
It seems that the CapRover deployment was configured to save all build artifacts, and Docker itself is [known for using large amounts of disk space](https://github.com/moby/moby/issues/21925).
Over the server's uptime of over 230 days, these Docker artifacts built up in `/var/lib/docker`, saturating the VPS disk.

### Impact
Between the hours of 20:00 and 21:50 (EDT) on August 31, developers and other Student Union members were unable to access CapRover and the StuyActivities API.
The MySQL database also crashed due to [OOM conditions](https://bugs.mysql.com/bug.php?id=104979).
No external reports are known.

### Detection
This incident was detected when Frank Wong noticed that he was unable to log into StuyActivities.

### Response
Frank informed the developer group chat, and attempted to revert backend code revisions to address this regression.
He then realized that the CapRover instance was offline, and went to investigate on the Digital Ocean dashboard.
Here, he discovered that the disk was full, and after looking into disk quota expansions, he attempted to access the server's login console.


### Recovery
Since the server password was not available at the time, the server was rebooted to reset the password.
a screen-share session to a login console was started in the group chat, where David Chen and Ben Pan joined to assist.

Due to the systemd-journalctl errors on the login console, along with a `du -sh /*` query revealing large storage use in `/var`,
the first course of action was to [purge journalctl logs](https://unix.stackexchange.com/questions/139513/how-to-clear-journalctl),
clearing up 2 GB of space, which was enough to restore the server to a usable state.

Afterwards, Frank authorized SSH keys for himself and David to conduct further maintenance on the server.

David continued to investigate the blockage in `/var`, and narrowed it down to `/var/lib/docker`, where `overlay2` data was taking up much of the space.
Thus, he ran [Docker volume cleanup commands](https://forums.docker.com/t/some-way-to-clean-up-identify-contents-of-var-lib-docker-overlay/30604/8),
along with container and image prune commands that were run by the previous maintainers, as recorded in the `.bash_history`.
This allowed the disk to go down to 47% usage, which was deemed to be enough.

While working on documentation and logs for this incident, a second incident was discovered: CapRover managed services were not functioning.

## Secondary Incident
### Fault
Due to the purge of Docker artifacts in the recovery of the primary incident, CapRover was unable to restore its configured application containers.

### Impact
All CapRover-managed services run on the Droplet server were offline.

### Detection
It was noticed that the `image-cdn` application, used for StuyActivities' club logos, was not functioning.

### Response
An update and reboot of the server was performed in the hope that applying updates would resolve the issue.

Later investigation of the CapRover deployments showed that the 502 Gateway Errors were due to the services being offline and their artifacts unavailable.

### Recovery
All CapRover-deployed services were inspected to ensure that they were online.
If they were not, the service would be rebuilt based off of the latest GitHub revisions.

However, the `image-cdn` service continued to crash in a loop due to an inability to connect to the `minio` service, which could not be redeployed due to it not being linked to a given repository.
Further investigation into the StuyActivities wiki documentation showed that the minio database was only used as an image cache, but no descriptions were given as to how to deploy it.

Eventually, the [CapRover "One-Click App" configuration used for the `minio` deployment](https://github.com/caprover/one-click-apps/blob/master/public/v4/apps/minio.yml) was discovered, 
and a manual application of its Dockerfile configuration restored full functionality.

## Recurrence
Based on the `.bash_history`, it can be presumed that Docker cleanups were necessary in the past too.

## Lessons Learned
- CapRover by default archives and depends on Docker artifacts
- Regular inspection and cleanup of logs is necessary to proactively investigate and prevent incidents
- Docker can be quite bloated in terms of disk usage
- One-click configurations were used in the setup of the server, and should be documented accordingly

## Corrective Actions
- Documentation was added to the `/root/` directory to provide guidance on this issue to future developers.
- A read-only script was added to display some commands that may be useful for mitigation of the issue.
- Further documentation is being added to the CapRover app descriptions, along with details in the StuyActivities wiki and this postmortem.

