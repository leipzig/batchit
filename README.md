batchit is a collection of utilities for working with AWS batch.


usage
=====

```
batchit Version: $version

ddv        : detach and delete a volume by id
ebsmount   : create and mount an EBS volume from an EC2 instance
efsmount   : EFS drive from an EC2 instance
localmount : RAID and mount local storage
submit     : run a batch command


```

submit
------

example:

```
batchit submit \
            --image worker:latest \
            --role worker-role \
            --queue big-queue \
            --jobname my-work \
            --cpus 32 \
            --mem 32000
            --envvars "sample=SS-1234" "reference=hg38" "bucket=my-s3-bucket" \
            --ebs /mnt/my-ebs:500:st1:ext4 \
            align.sh
```

### Interactive

To get an interactive job, use the `submit` command, but instead of a script (`align.sh`) above,
use, for example, "interactive:20" to get an interactive job that will run for 20 minutes.

This command will start a job that sleeps for 20 minutes and the output an ssh command that will drop
the user into the docker container running that command.

This is useful for debugging as it quickly drops a user into the same environment that the jobs
will be run in.

#### batchit requirements

#### AWS
AWS Batch itself requires the `AWSBatchServiceRole` and `ecsInstanceRole` generated by running the [first-run wizard](https://console.aws.amazon.com/batch/home#/wizard). Because batchit containers use EC2 and S3 services, batchit requires an [ecsTaskRole](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_IAM_role.html) with `AmazonEC2FullAccess` and `AmazonS3FullAccess`. This is the `worker-role` above.


#### Docker
The `image` must be present in your elastic container registry and the container will itself need batchit as a dependency if `--ebs` is used. A typical Dockerfile entry for this will look like:

```
RUN apt-get install -y wget
RUN wget -qO /usr/bin/batchit https://github.com/base2genomics/batchit/releases/latest
RUN chmod +x /usr/bin/batchit
```

In this example `align.sh` contains the commands to be run. It will have access to a 500GB
`st1` volume created with ext4 and mounted to `/mnt/my-ebs`. This will automatically set docker to run in privileged
mode so that it has access to the EBS volume that is attached to /dev in the instance.
(We stole this idea from [aegea](https://github.com/kislyuk/aegea))

The volume will be cleaned up automatically when the **container** exits.

Note that array jobs are also supported with `--arraysize INT` parameter. Currently, the user is responsible for specifying
the dependency mode (`N_TO_N` or `SEQUENTIAL`) to the `--dependson` parameter.

For this example a simplified `align.sh` might look like (always include the first two lines):

```
set -euo pipefail
cd $TMPDIR
aws s3 cp s3://${bucket}/${sample}_r1.fq .
aws s3 cp s3://${bucket}/${sample}_r2.fq .
aws s3 sync s3://${bucket}/assets/${reference} .
bwa mem -t ${cpus} ${reference}.fa ${sample}_r1.fq ${sample}_r2.fq \
      | samtools sort -o ${sample}.bam
samtools index ${sample}.bam
aws s3 cp ${sample}.bam s3://${bucket}/
aws s3 cp ${sample}.bam.bai s3://${bucket}/
```

ebsmount
--------

create, attach, format, and mount an EBS volume of the specified size and type to the specified mount-point.
If `-n` is greater than 1, then it will automatically RAID0 (performance, not reliability) the drives.
This is used (in shorthand) by the `--ebs` argument to `batchit submit` above.

```
Usage: batchit [--size SIZE] --mountpoint MOUNTPOINT [--volumetype VOLUMETYPE] [--fstype FSTYPE] [--iops IOPS] [--n N] [--keep]

Options:
  --size SIZE, -s SIZE   size in GB of desired EBS volume [default: 200]
  --mountpoint MOUNTPOINT, -m MOUNTPOINT
                         directory on which to mount the EBS volume
  --volumetype VOLUMETYPE, -v VOLUMETYPE
                         desired volume type; gp2 for General Purpose SSD; io1 for Provisioned IOPS SSD; st1 for Throughput Optimized HDD; sc1 for HDD or Magnetic volumes; standard for infrequent [default: gp2]
  --fstype FSTYPE, -t FSTYPE
                         file system type to create (argument must be accepted by mkfs) [default: ext4]
  --iops IOPS, -i IOPS   Provisioned IOPS. Only valid for volume type io1. Range is 100 to 20000 and <= 50\*size of volume.
  --n N, -n N            number of volumes to request. These will be RAID0'd into a single volume for better write speed and available as a single drive at the specified mount point. [default: 1]
  --keep, -k             dont delete the volume(s) on termination (default is to delete)
  --help, -h             display this help and exit
  --version              display version and exit

```

efsmount
--------

This is a trivial wrapper around mounting an EFS volume.

```
Usage: batchit [--mountoptions MOUNTOPTIONS] EFS MOUNTPOINT

Positional arguments:
  EFS                    efs DNS and mount path (e.g.fs-XXXXXX.efs.us-east-1.amazonaws.com:/mnt/efs/)
  MOUNTPOINT             local directory on which to mount the EBS volume

Options:
  --mountoptions MOUNTOPTIONS, -o MOUNTOPTIONS
                         options to send to mount command
  --help, -h             display this help and exit
```
