# Phone integration handoff

This branch adds phone control without changing the RX motion, laser, job
executor, or TX bridge implementation.

## Embedded side

The phone-enabled RX build contains these source/config changes:

- `src/ws63_laser_rx_unified/routes/sle_job/sle_job_route_server.c`
- `src/ws63_laser_rx_unified/Kconfig`
- `configs/ws63_rx_unified_defconfig`

The defconfig enables:

```text
CONFIG_LASER_RX_SLE_JOB_ALLOW_PHONE=y
```

With this option enabled, RX accepts a non-fixed SLE peer for the phone. The
existing single-owner rule is unchanged: the first peer that writes the job
data characteristic owns control; other writers are dropped. Disconnecting the
owner uses the existing safe-stop callback.

One phone connection is admitted at a time. The phone path negotiates a
512-byte link data length so STATUS responses and upload chunks fit, while it
keeps the phone-compatible/default PHY and MCS parameters.
The fixed TX and Screen paths retain their existing 512-byte, PHY 4M, and MCS10
throughput tuning. Notification client-configuration writes are handled as
SSAP control traffic and cannot claim job ownership.

Build the RX package in the teammate's WSL environment with the normal RX
workflow. The expected package is:

```text
ws63-liteos-app_rx_unified_all.fwpkg
```

After flashing the package, the startup line must contain:

```text
phone_integration=phone-rx-v7-20260719 ... phone_data_len=512 phone_phy_mcs_tune=0
```

When the phone connects, RX must log `accept Phone`, a successful Phone
`data_len=512`, and `[job_rx_link_tune] skip Phone`; it must not log a forced
Phone PHY 4M/MCS10 change.
The notification subscription should appear once as `[RX_CCCD]` and must not
enter the job parser. A disconnect is labelled with `reason` and `source` so
the remote/local initiator is explicit; timeout diagnosis still uses the phone
HiLog together with this RX log.

## Phone app side

Copy these files from the HarmonyOS phone project into the matching project
paths before building the HAP:

- `entry/src/main/ets/pages/SsapClientPage.ets`
- `entry/src/main/ets/protocol/JobProtocol.ets`

The client connects to the RX SLE service and uses the existing service and
characteristic UUIDs. It supports G-code upload, uploaded-job start, status
monitoring, stop/resume/abort, focus control, and route/status operations.

## Runtime ownership order

Use the phone as the control owner when TX is not already the owner. If TX has
already written a packet, disconnect TX or wait for the owner-safe-stop path
before starting a phone job. RX must report the phone connection and the phone
must receive ACK/STATUS notifications before uploading a real job.

Do not flash the TX or Screen package onto RX. Use only the RX package named
above for the RX board.
