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

Build the RX package in the teammate's WSL environment with the normal RX
workflow. The expected package is:

```text
ws63-liteos-app_rx_unified_all.fwpkg
```

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
