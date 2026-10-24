# 00 --- Prerequisites and AWS EC2 Creation

## Purpose

This is the prerequisite lab before installing Kubernetes tools. It
explains what is needed, how to select a suitable EC2 instance for this
learning cluster, and how to connect safely.

> The steps below document the decisions and environment known from this
> lab. Exact AWS console labels, prices, instance availability, and
> free-tier/credit terms may change. Confirm them in the current AWS
> console. The prior session does not preserve every exact click or
> selected option, so console choices not known for certain are
> identified as recommendations rather than claimed historical facts.

## 1. What you need before starting

-   An AWS account with access to launch EC2 instances.
-   A local computer with an SSH client. Windows PowerShell and Git Bash
    both provide SSH in common installations.
-   A key pair for SSH access, stored securely on your local computer.
-   A security group that permits SSH (TCP 22) only from your trusted
    public IP where practical.
-   A plan to stop or terminate resources when finished, and to review
    AWS billing.
-   Basic terminal familiarity.

Never upload your `.pem` private key, AWS access keys, passwords, or
kubeconfig credentials to GitHub.

## 2. Understand the resources

-   **EC2 instance:** the virtual machine.
-   **AMI:** the operating-system image. This lab used Red Hat
    Enterprise Linux 10.2.
-   **Instance type:** the CPU and memory size.
-   **EBS volume:** persistent block storage attached to the instance.
-   **Key pair:** allows SSH authentication.
-   **Security group:** acts as a virtual firewall for network traffic.
-   **Public IPv4/DNS:** can be used to connect over the internet if
    assigned and permitted by routing/security rules.

## 3. Select CPU and memory

The instance used in this lab had **2 vCPUs and approximately 3.5 GiB
RAM**. It was enough to run Docker, Minikube, Kubernetes system
components, and a small NGINX Pod, but memory was tight.

Minikube was configured with 3072 MiB of memory, leaving limited
overhead for RHEL, Docker, SSH, and other processes. The startup output
warned that the host could become unstable.

### Practical sizing guidance

  -----------------------------------------------------------------------
  Intended use            Suggested starting      Notes
                          point                   
  ----------------------- ----------------------- -----------------------
  `kubectl` client only,  Small instance or local `kubectl` alone does
  no local cluster        computer                not run Kubernetes.

  Basic Minikube test     2 vCPUs and 4 GiB RAM   Monitor memory and
  with one small workload can work, but is tight  avoid many workloads.

  More comfortable        2--4 vCPUs and 8 GiB    More headroom; check
  learning with multiple  RAM if budget permits   current EC2 pricing.
  Pods and add-ons                                

  Larger multi-service or Size based on workload; This is not production
  load testing            consider a dedicated    sizing guidance.
                          Kubernetes learning     
                          environment             
  -----------------------------------------------------------------------

These are practical learning estimates, not guarantees or exact AWS
recommendations. Check available instance types and costs in your
account's region. Do not assume an instance is free because credits or
free-tier offers are present.

## 4. Launch an EC2 instance --- recommended sequence

1.  Open the AWS Management Console and navigate to **EC2 → Instances →
    Launch instances**.
2.  Enter a clear name, such as `k8s-learning-lab`.
3.  Select an AMI. The recorded lab uses **Red Hat Enterprise Linux 10.2
    (x86_64)**. Use a supported image available in your region; package
    repositories and module availability can differ by image.
4.  Select an instance type. The recorded instance had 2 vCPUs and about
    3.5 GiB RAM. For a new lab, consider 8 GiB RAM if affordable;
    otherwise start with the smaller instance and reduce Minikube's
    memory.
5.  Select or create a key pair. Download the private key once and keep
    it outside the repository.
6.  Configure networking and a security group. Permit SSH/TCP 22 from
    **My IP** or another trusted IP range. Do not use `0.0.0.0/0` for
    SSH unless there is a specific, temporary reason and you understand
    the exposure.
7.  Configure the root EBS volume. This lab began with a 10 GiB root
    disk and later increased it to 25 GiB. For container images and
    Kubernetes components, 25 GiB is a more comfortable learning
    starting point than 10 GiB, though disk use depends on images and
    logs.
8.  Review the estimated cost and configuration, then launch.
9.  Wait until the instance is running and its status checks pass.
10. Record the public DNS/IP for SSH use, but avoid publishing it if
    unnecessary.

The exact original console choices for every field were not captured.
This sequence is a reproducible recommendation based on the environment
used.

## 5. Connect to the instance

From PowerShell or Git Bash, go to the folder containing your private
key and run:

``` bash
ssh -i "K8s.pem" ec2-user@YOUR_EC2_PUBLIC_DNS
```

Replace `YOUR_EC2_PUBLIC_DNS` with the instance's current public DNS
name. The username `ec2-user` is used for the RHEL image in this lab.
Other AMIs may use a different username.

Once connected, verify the host:

``` bash
whoami
hostname
cat /etc/redhat-release
uname -r
nproc
free -h
df -h /
```

Observed environment: RHEL 10.2, x86_64, 2 vCPUs, around 3.5 GiB RAM,
and a root disk expanded to 25 GiB.

## 6. Expand the root EBS volume from 10 GiB to 25 GiB

This lab's EBS root volume was increased from 10 GiB to 25 GiB.
Increasing the EBS volume size in AWS does not always expand the
partition and filesystem automatically; both layers may need expansion.

**Before changing disks:** confirm the device, partition, filesystem
type, and current layout. Device names can differ. The commands below
reflect the layout observed in this lab and should not be blindly copied
to a different disk.

In the EC2 console: 1. Open **EC2 → Volumes**. 2. Identify the root EBS
volume attached to the correct instance. 3. Choose **Modify volume**,
change the size from 10 GiB to 25 GiB, and confirm. 4. Wait for the
modification to progress sufficiently, then inspect from the OS.

On the instance, the observed layout used `/dev/nvme0n1` with partition
3 and XFS:

``` bash
lsblk
df -hT /
```

After verifying that your own device/partition layout matches, the
commands used in this lab were:

``` bash
sudo growpart /dev/nvme0n1 3
sudo xfs_growfs /
df -hT /
```

-   `growpart` grows partition 3 to use the available space on the
    device.
-   `xfs_growfs /` expands the mounted XFS filesystem to use the
    enlarged partition.
-   `df -hT /` verifies the resulting filesystem size and type.

If `growpart` or `xfs_growfs` is unavailable or reports a different
layout, stop and inspect `lsblk` and `df -hT`; do not guess a partition
number. Keep important data backed up before disk changes.

## 7. Cost and security checklist

-   Check the AWS billing dashboard and current pricing before launch.
-   Stop the instance when not using it if you want to reduce compute
    charges; attached storage can still incur charges.
-   Terminate the instance only when you are sure you no longer need it;
    termination can delete root storage depending on its configuration.
-   Restrict SSH to trusted IP addresses.
-   Do not expose Kubernetes ports or port-forwarding ports publicly for
    this lab.
-   Keep private keys outside the Git repository and verify `.gitignore`
    before pushing.
