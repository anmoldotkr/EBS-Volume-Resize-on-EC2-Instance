# EC2 EBS Volume Resize Procedure and DevOps Interview Questions

## Part 1: EBS Volume Resize Procedure

### Objective
Increase the root EBS volume capacity of an Ubuntu EC2 instance to 300 GiB while preserving existing data.

### Step 1: Increase the EBS Volume in AWS Console

1. Log in to the AWS Management Console.
2. Navigate to **EC2 → Instances**.
3. Select the target EC2 instance.
4. Open the **Storage** tab and click the attached EBS Volume ID.
5. On the Volumes page, select the volume.
6. Choose **Actions → Modify volume**.
7. Increase the volume size to 300 GiB.
8. Click **Modify** and confirm.
9. Monitor the volume modification status and wait until it is in a state that permits filesystem expansion.

**Important:** Increasing the EBS volume size does not automatically expand the partition or filesystem inside Ubuntu.

### Step 2: Connect to EC2 and Inspect the Disk

Connect to the instance using SSH and run:

```bash
lsblk
df -hT /
```

Identified the root disk as `/dev/nvme0n1`, with a 300 GB disk and an approximately 200 GB root partition.

- `lsblk` displays disks, partitions, and mount points.
- `df -hT /` displays root filesystem capacity, usage, and filesystem type.

### Step 3: Inspect the Partition Table

Run:

```bash
sudo parted /dev/nvme0n1 print free
```

A GPT warning indicated that the partition table did not recognize the full disk size. Selected `Fix` to update the GPT partition table.

The output showed approximately 107 GB of unallocated space after the root partition.

### Step 4: Extend the Root Partition

Run:

```bash
sudo growpart /dev/nvme0n1 1
```

This expanded partition 1 to use the available space.

### Step 5: Expand the Filesystem

The root filesystem was identified as ext4. Run:

```bash
sudo resize2fs /dev/nvme0n1p1
```

This expanded the ext4 filesystem to use the newly available partition capacity.

### Step 6: Verify the Changes

Run:

```bash
lsblk
df -h
```

Confirmed that the root partition and filesystem had expanded successfully.

### Final Result

| Metric | Before | After |
|---|---:|---:|
| EBS volume size | 300 GiB | 300 GiB |
| Root filesystem size | 194 GiB | 291 GiB |
| Used space | 122 GiB | 122 GiB |
| Available space | 73 GiB | 170 GiB |
| Disk usage | 63% | 42% |

**Status:** Successfully expanded the root filesystem without data loss or a reboot.

---

## Part 2: DevOps Engineer Interview Questions and Answers

### Q1. What is an EBS volume in AWS?

**Answer:** Amazon Elastic Block Store (EBS) provides persistent block storage for EC2 instances. It is commonly used for operating system disks, application data, and databases. EBS volumes persist independently of the lifecycle of an EC2 instance, subject to the volume and instance configuration.

### Q2. What is the difference between a disk, partition, and filesystem?

**Answer:**

- **Disk:** The complete block storage device, such as `/dev/nvme0n1`.
- **Partition:** A defined region of the disk, such as `/dev/nvme0n1p1`.
- **Filesystem:** The structure used to store and organize files, such as ext4 or XFS.

Increasing the disk size alone does not necessarily expand the partition or filesystem.

### Q3. Why did you use `lsblk`?

**Answer:** I used `lsblk` to inspect the block devices attached to the EC2 instance, their sizes, partitions, and mount points. It helped me identify the 300 GiB root disk and its root partition before making changes.

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

### Q4. Why did you use `df -hT`?

**Answer:** `df -hT` displays mounted filesystem capacity, used space, available space, usage percentage, and filesystem type. I used it to verify the actual usable root filesystem capacity before and after resizing.

**Key difference:** `lsblk` shows block-device layout, whereas `df` shows filesystem usage.

### Q5. Why did you use `parted`?

**Answer:** I used `parted` to inspect the partition table and identify unallocated space on the disk.

```bash
sudo parted /dev/nvme0n1 print free
```

It also displayed a warning that the GPT partition table did not use the full disk capacity. I selected `Fix` to update the GPT metadata.

`parted` is a disk partitioning utility; it can inspect and modify partition tables. In this operation, I used it to inspect the layout and resolve the GPT size warning.

### Q6. What is GPT, and why did it show a warning?

**Answer:** GPT (GUID Partition Table) is a partition-table format used to describe disk partitions. When an EBS volume is enlarged, the backup GPT header can remain at the previous end of the disk. The partitioning tool may then warn that the table does not use the full device size.

Selecting `Fix` updates the GPT metadata to recognize the expanded disk capacity. This does not, by itself, expand the root partition or filesystem.

### Q7. Why did you use `growpart` after `parted`?

**Answer:** `parted` helped me inspect the disk layout and fix the GPT warning. I then used `growpart` to expand the existing root partition into the adjacent unallocated space.

```bash
sudo growpart /dev/nvme0n1 1
```

The arguments mean:

- `/dev/nvme0n1`: The disk device.
- `1`: The partition number to grow.

`growpart` is specifically designed to grow an existing partition to use available space. It does not create a new filesystem or format the partition.

### Q8. Why did you use `resize2fs`?

**Answer:** The root filesystem was ext4. After expanding the partition, I used `resize2fs` to expand the ext4 filesystem so the operating system could use the additional capacity.

```bash
sudo resize2fs /dev/nvme0n1p1
```

Expanding a partition and expanding its filesystem are separate operations.

### Q9. What if the filesystem is XFS instead of ext4?

**Answer:** For XFS, the filesystem is grown using `xfs_growfs` on the mounted filesystem, rather than `resize2fs`.

For example, if the XFS filesystem is mounted at `/`:

```bash
sudo xfs_growfs /
```

Always verify the filesystem type before choosing the resize command.

### Q10. Can you increase an EBS volume without stopping the EC2 instance?

**Answer:** In many cases, yes. EBS Elastic Volumes supports increasing the size of supported attached volumes while the instance is running. The operating system may still need partition and filesystem expansion afterward.

Compatibility, volume state, and operating system configuration should be checked before proceeding.

### Q11. Why did you not create a new partition?

**Answer:** The existing root partition had unallocated space immediately after it. Therefore, I expanded the existing root partition rather than creating a separate partition. This allowed the root filesystem to use the additional capacity without changing the mount layout.

### Q12. Is a reboot required after increasing an EBS volume?

**Answer:** Usually not for supported online EBS volume expansion and filesystems. In this case, the partition and ext4 filesystem were expanded online, and the new capacity was available without a reboot.

### Q13. What precautions should you take before resizing a production EBS volume?

**Answer:**

1. Take or verify a recent EBS snapshot.
2. Confirm the correct EC2 instance and volume ID.
3. Inspect the disk and partition layout using `lsblk`.
4. Verify the filesystem type using `df -hT`.
5. Confirm that the target partition can safely be expanded.
6. Execute the resize commands carefully.
7. Verify the final filesystem capacity and application health.

EBS volume increases are generally not reversible by simply reducing the volume size. Restoring to a smaller volume may require creating a new volume and migrating the data.

### Q14. What would you do if `growpart` returned `NOCHANGE`?

**Answer:** I would inspect the partition table and disk layout again:

```bash
lsblk
sudo parted /dev/nvme0n1 print free
```

Possible reasons include the partition already occupying the available space, the wrong partition number being supplied, or the partition table not yet reflecting the expanded disk. I would confirm the root disk and available space before retrying.

### Q15. How did you verify that the operation succeeded?

**Answer:** I verified the disk and filesystem using:

```bash
lsblk
df -h
```

The root filesystem increased from approximately 194 GiB to 291 GiB, available space increased from 73 GiB to 170 GiB, and usage dropped from 63% to 42%. The existing data remained intact.

---

## Part 3: Key Commands to Remember

| Command | Purpose |
|---|---|
| `lsblk` | Inspect disks and partitions |
| `df -hT /` | Check root filesystem size and type |
| `sudo parted /dev/nvme0n1 print free` | Inspect partition layout and free space |
| `sudo growpart /dev/nvme0n1 1` | Expand partition 1 |
| `sudo resize2fs /dev/nvme0n1p1` | Expand an ext4 filesystem |
| `sudo xfs_growfs /` | Expand a mounted XFS root filesystem |
| `sudo fdisk -l` | Inspect disk and partition information |

**Interview takeaway:** Always remember the sequence: **Increase EBS volume size → Inspect the disk → Expand the partition → Expand the filesystem → Verify the result.**
