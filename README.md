# EBS-Volume-Resize-on-EC2-Instance

# EC2 EBS Volume Resize Procedure

## Objective
Increase the root EBS volume capacity of an Ubuntu EC2 instance from approximately 200 GB to 300 GB while preserving existing data.

## Step 1: Increase EBS Volume Size in AWS Console

1. Log in to the **AWS Management Console**.
2. Navigate to **EC2 → Instances**.
3. Select the target EC2 instance.
4. Open the **Storage** tab and click the attached EBS Volume ID.
5. On the **Volumes** page, select the volume.
6. Choose **Actions → Modify volume**.
7. Increase the volume size to **300 GiB**.
8. Click **Modify** and confirm the change.
9. Check the volume modification status. Wait until the modification is completed or the volume is in a state that permits expansion.

Note: Increasing the EBS volume size does not automatically expand the partition or filesystem inside Ubuntu.

## Step 2: Connect to EC2 Through SSH

Connect to the target Ubuntu EC2 instance and verify the disk layout:

```bash
lsblk
df -hT /
```

Identified the root disk as `/dev/nvme0n1`, with a 300 GB disk and an approximately 200 GB root partition.

## Step 3: Update the GPT Partition Table

Check the partition layout and unallocated space:

```bash
sudo parted /dev/nvme0n1 print free
```

A warning indicated that the GPT partition table did not recognize the full disk size. Selected `Fix` to update the partition table.

Afterward, approximately 107 GB of unallocated space was visible.

## Step 4: Extend the Root Partition

Expanded partition 1 to use the available space:

```bash
sudo growpart /dev/nvme0n1 1
```

## Step 5: Extend the Filesystem

The root filesystem was identified as ext4. Expanded it using:

```bash
sudo resize2fs /dev/nvme0n1p1
```

## Step 6: Verify the New Capacity

Ran the following commands:

```bash
lsblk
df -h
```

Confirmed that the root partition and filesystem had expanded successfully.

## Final Result

| Metric | Before | After |
|---|---:|---:|
| EBS volume size | 300 GB | 300 GB |
| Root filesystem size | 194 GB | 291 GB |
| Used space | 122 GB | 122 GB |
| Available space | 73 GB | 170 GB |
| Disk usage | 63% | 42% |

## Conclusion

The EBS volume was configured at 300 GiB, and the Ubuntu root partition and ext4 filesystem were successfully expanded to use the additional storage.

- Existing data was preserved.
- The EFI partition remained unchanged.
- No reboot was required.
- The additional disk capacity became available immediately.

**Status: Completed successfully.**
