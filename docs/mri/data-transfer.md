# Data Transfer (Stage 08)

Stage 08 transfers the completed BIDS dataset from the processing cluster to a remote destination using the Globus file transfer service. Globus provides reliable, resumable, high-performance data transfer between institutional storage endpoints without requiring the user to maintain an active session during the transfer.

## Prerequisites

Before running the transfer:

1. **Globus CLI** must be installed and authenticated in your environment.
2. **Source and destination endpoints** must be configured and accessible.
3. **Endpoint permissions** must allow read access on the source and write access on the destination.

### Globus CLI Setup

Install the Globus CLI in a conda environment or via pip:

```bash
pip install globus-cli
```

Authenticate with your institutional credentials:

```bash
globus login
```

This opens a browser-based authentication flow. After completing it, the CLI stores credentials locally for subsequent commands.

### Endpoint Configuration

Identify the UUIDs of your source and destination Globus endpoints. You can search for endpoints by name:

```bash
globus endpoint search "OpenMind"
globus endpoint search "Engaging"
```

Configure the endpoints in `config.sh` or directly in the transfer script:

```bash
export TRANSFER_SOURCE_ENDPOINT="your-source-endpoint-uuid"
export TRANSFER_DEST_ENDPOINT="your-destination-endpoint-uuid"
export TRANSFER_SOURCE_DIR="${BIDS_DIR}"
export TRANSFER_DEST_DIR="/remote/path/to/destination"
```

The default transfer script (`08_data_transfer/Transfer_BIDS.sh`) contains hardcoded endpoint IDs and paths from the original processing environment. Update these values before running.

## Run Command

```bash
bash 08_data_transfer/Transfer_BIDS.sh
```

The script performs the following steps:

1. Scans `SOURCE_DIR` for all `sub-*` directories
2. Writes a batch file (`transfer_batch.txt`) listing each subject directory as a recursive transfer entry
3. Submits the batch transfer to Globus using `globus transfer`
4. Prints the task ID for monitoring

The batch file format is one line per subject directory:

```
/source/path/sub-00123456 /dest/path/sub-00123456 --recursive
/source/path/sub-00999999 /dest/path/sub-00999999 --recursive
```

## SLURM Resources

The transfer script includes SLURM directives for submission as a batch job:

| Parameter | Value |
|-----------|-------|
| Cores | `32` |
| Memory | `128 GB` |
| Wall time | `47:00:00` |
| GPU | `1x A100` (from original script; remove if not needed) |

!!! tip "GPU request"
    The original script includes `--gres=gpu:a100:1`, which was specific to the original cluster's queue configuration. If your cluster does not require a GPU allocation to access the necessary storage or network, remove this line from the SLURM directives to avoid unnecessary resource contention.

## Verification

After the transfer is submitted, the script prints a task ID. Use this to monitor progress:

```bash
# Check transfer status
globus task show <task_id>

# List recent transfers
globus task list

# Wait for completion (blocking)
globus task wait <task_id>
```

The `globus task show` output includes:

- Transfer status (ACTIVE, SUCCEEDED, FAILED)
- Number of files transferred and remaining
- Bytes transferred
- Error messages (if any)

After the transfer completes, verify the destination:

1. Confirm the number of subject directories matches the source:
    ```bash
    # On source
    ls -d ${TRANSFER_SOURCE_DIR}/sub-* | wc -l

    # On destination (if accessible)
    ls -d ${TRANSFER_DEST_DIR}/sub-* | wc -l
    ```

2. Spot-check a few subjects for completeness by comparing file counts between source and destination.

## Common Issues

**Authentication failure.** If `globus transfer` returns an authentication error, re-authenticate:

```bash
globus login --force
```

On headless systems where a browser is not available, use the `--no-local-server` flag:

```bash
globus login --no-local-server
```

This provides a URL to paste into a browser on another machine, then prompts for an authorization code.

**Endpoint not found or access denied.** Verify that both endpoints are active and that your Globus identity has the required permissions. Some institutional endpoints require activation before use:

```bash
globus endpoint activate <endpoint_id>
```

**Transfer stalls or fails partway through.** Globus transfers are automatically resumable. If a transfer fails, resubmit it; Globus will skip files that have already been transferred successfully. Check the error details:

```bash
globus task event-list <task_id>
```

**Destination path does not exist.** Globus does not create intermediate directories. Ensure the destination path (`TRANSFER_DEST_DIR`) exists on the destination endpoint before initiating the transfer.

**Incorrect source or destination paths.** The default script uses hardcoded paths from the original environment:

```bash
SOURCE_DIR="/om2/user/mabdel03/files/Ravi_ISO_MRI/reformatted"
DEST_DIR="/pool001/mabdel03/BIDS_MRI_Data"
```

Update these to match your source BIDS directory and intended destination before running.
