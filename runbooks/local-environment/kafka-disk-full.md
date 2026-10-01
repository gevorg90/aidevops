# Local environment: Kafka failure after disk exhaustion

## Incident: 2026-10-01

- SSH host: `jnk` (`192.168.0.44`).
- Service: `kafka.service`, running as user `kafka`.
- Broker endpoint: `kk.local.renderforest.com:9092`.
- Configuration: `/opt/kafka/config/server.properties`.
- Data directory: `/data/kafka` on the shared `/data` filesystem.
- Mode: KRaft, with `process.roles=broker,controller` and `node.id=1`. ZooKeeper is not a dependency of this Kafka instance.

All incident times below are host local time (UTC+04:00).

## Symptoms and cause

Kafka exited with status `1` at 13:29:25 after running since August 22. Logs at 13:29:24 showed:

```text
ERROR Error while writing to checkpoint file /data/kafka/replication-offset-checkpoint
java.io.IOException: No space left on device
ERROR Shutdown broker because all log dirs in /data/kafka have failed
```

The immediate cause was disk exhaustion during a checkpoint write. Kafka marked its only log directory as failed and shut down.

At investigation time, `/data` already had 456 GB available and was 74% used. Inodes were only 1% used. The investigation did not establish what freed the space between the failure and the checks.

`/data/nexus` accounted for approximately 1.3 TB on the shared filesystem; `/data/kafka` accounted for approximately 89 MB. These measurements identify the largest current consumer, but do not prove which workload exhausted the disk at the failure time.

## Troubleshooting procedure

Connect using the existing SSH alias:

```sh
ssh jnk
```

Inspect the service and failure logs before changing anything. Journal access requires sudo on this host.

```sh
systemctl status kafka --no-pager -l
systemctl show kafka -p ActiveState -p SubState -p Result -p ExecMainStatus
sudo -n journalctl -u kafka -n 100 --no-pager
tail -n 80 /opt/kafka/logs/server.log
```

For this incident, the precise failure window was:

```sh
sudo -n journalctl -u kafka \
  --since '2026-10-01 13:29:00' \
  --until '2026-10-01 13:30:00' --no-pager
```

Check the filesystem containing Kafka data, inode availability, and the largest consumers:

```sh
df -h /data/kafka /opt/kafka
df -i /data/kafka
free -h
sudo -n du -xhd1 /data | sort -h
```

If space is still exhausted, establish a concrete remediation before starting Kafka. File or directory deletion requires explicit confirmation under the workspace AGENTS.md. Do not manually remove Kafka partition data or format Kafka storage as a recovery step.

## Recovery performed

With disk space available, the only service change was:

```sh
sudo -n systemctl start kafka
```

The service process started at 14:09:07. Kafka recovered its logs and reported `Kafka Server started` at 14:10:00. Early metadata checks failed while recovery was still in progress; subsequent checks succeeded.

No configuration changes, manual data deletion, storage formatting, or ZooKeeper restart were performed. A diagnostic stderr file was written to `/tmp/kafka-health-check.err` during verification.

## Verify recovery

An active systemd process alone does not establish Kafka health. Allow log recovery to finish, then run application checks against the advertised endpoint:

```sh
systemctl is-active kafka
timeout 30 /opt/kafka/bin/kafka-broker-api-versions.sh \
  --bootstrap-server kk.local.renderforest.com:9092
timeout 30 /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server kk.local.renderforest.com:9092 --list
timeout 30 /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server kk.local.renderforest.com:9092 \
  --describe --unavailable-partitions
ss -lnt | grep -E ':9092|:9093'
df -h /data
```

Each Kafka command must exit successfully. The unavailable-partitions check should return no partitions. If a command times out or fails, inspect startup logs and repeat only after resolving the cause or confirming recovery is progressing.

Verified results for this incident:

- Service: `active (running)`.
- Broker API: successful response from broker `1`, `isFenced: false`.
- Topic metadata: 112 topics returned successfully.
- Unavailable partitions: none; command exited `0`.
- Listening ports: `9092` and `9093`.
- `/data`: 456 GB available, 74% used.

These checks verified broker and metadata availability. No application produce/consume round trip was performed.

## Follow-up

Monitor free space on the shared `/data` filesystem and investigate Nexus storage growth and existing cleanup policies. Determine what freed space after the failure before attributing the incident to a particular workload. Any cleanup or configuration change should be reviewed separately with the required authorization.
