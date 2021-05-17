_DRAFT blog post for Enable Sysadmin_

----

# To swap, or not to swap, that is the cloud question

![matthew-waring-MJAoiige14E-unsplash.jpg](matthew-waring-MJAoiige14E-unsplash.jpg)

If you want to start an argument with a Linux user, ask about swap memory. Some
praise it as a cushion or a safety net while others disparage it as a crutch and
a destroyer of system performance. [Born in the 1960's], swap memory evolved
over the years on Linux to serve two important functions:

  * Emergency memory when a system consumes all of its RAM and still needs more
  * Parking lot for rarely-used memory pages that take up valuable system RAM

Many Linux distributions, including Red Hat, [recommend swap memory] for all
systems. However, if you take a look at the majority of cloud instances from
various distributions, you find that swap memory is absent.

Keep reading to find out why. 🤔

[Born in the 1960's]: https://en.wikipedia.org/wiki/Memory_paging#History
[recommend swap memory]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_storage_devices/getting-started-with-swap_managing-storage-devices

## Overview of swap

Swap exists on most systems as a partition on a disk. After partitioning,
administrators format the partition with `mkswap`, enable it with `swapon`, and
the kernel instantly sees available swap memory. Systems without an available
partition can use a swap file, which is just a file on an existing filesystem
that is formatted with `mkswap` and enabled.

Both methods work well, but putting swap on a partition leads to better
performance since you skip the overhead of a swap file on an existing partition.

When system RAM is scarce, Linux can store memory pages in swap to avoid killing
processes and crashing the system. Disks are much slower than system RAM and
this reduces system performance until system RAM is freed. If memory usage
continues climbing to the point where system RAM and swap are fully exhausted,
the OOM killer appears and begins killing processes until enough RAM is
available.

The long-standing advice on swap size is double the size of your system's RAM.
For example, allocate 2GB of swap for systems with 1GB of system RAM. Although
this ratio works well for smaller systems, it doesn't scale up for systems with
hundreds of gigabytes of system RAM.

## The case for swap on cloud

The rise of microservices (small, interconnected services that form a large
application) led companies to deploy a larger number of smaller instances.
Smaller instances come with less system RAM and this increases the risk of the
dreaded [OOM killer] showing up to kill processes until the system has enough
free memory.  Adding swap to these systems helps in two ways.

First, processes can briefly burst out of the system RAM into swap during
periods of high load. Administrators learn about these events from their
monitoring systems and they have time to investigate the reason for the burst
while it happens on the system. In the case of memory leaks, developers can
inspect the process to understand what went wrong while it's still alive. This
can also be a signal that administrators need larger instance types for their
application as it grows.

Second, the Linux kernel keeps watch over rarely used memory pages and sends
them out to swap to preserve precious system RAM. The sysctl setting
`vm.swappiness` controls the kernel's desire to push memory pages to swap. This
could help a cloud instance by keeping the most active pages in system RAM while
pushing rarely used pages out to swap memory.

[OOM killer]: https://en.wikipedia.org/wiki/Out_of_memory

## The case against swap on cloud

The ["pets vs cattle" analogy] comes up frequently when talking about cloud
instances and administrators banish swap since they can replace a misbehaving
instance with a new one automatically. Over time, their monitoring metrics show
an increase in instances needing replacement due to OOM killer taking their
application offline. The solution could be larger instances or further
investigation of memory leaks under load. Systemd will also restart these
processes quickly when possible.

Choosing a swap memory location on cloud instances requires more thought and
planning. On physical servers, swap on a locally attached NVME disk is fast
enough, but what about a cloud instance with external storage mounted, such as
AWS' Elastic Block Storage (EBS)? Performance on EBS varies depending on which
type of EBS you choose and your neighbors on the system. Swap on a remote
storage system could lead to poor instance performance when system overflows its
RAM. Administrators may opt to omit swap and replace these instances when they
overflow their RAM instead of wrestling with poorly performing server that is
still handling requests.

Finally, many cloud instances become part of Kubernetes and OpenShift clusters,
and they have a challenge when dealing with swap memory. There is a
[long-running GitHub issue] about how to handle swap memory appropriately since
it throws off the memory accounting for containers. Workarounds exist, but swap
memory generally gets skipped on these systems.

["pets vs cattle" analogy]: http://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/
[long-running GitHub issue]: https://github.com/kubernetes/kubernetes/issues/53533

## Deploying cloud instances with swap

Should you decide that provisioning swap memory benefits your applications,
cloud-init gives you the ability to provision swap memory on the first boot via
the [`mounts` module]. Just add a few lines to your existing cloud-config user
data:

```yaml
swap:
    filename: /swapfile
    size: auto
    maxsize: 4294967296
```

This configuration tells cloud-init to create a swap file at `/swapfile` with an
automatic size that is equal to or less than 4 GiB. The formula for automatic swap sizing is in [cloud-init's source code]:

```python
formulas = [
    # < 1G: swap = double memory
    (1 * GB, lambda x: x * 2),
    # < 2G: swap = 2G
    (2 * GB, lambda x: 2 * GB),
    # < 4G: swap = memory
    (4 * GB, lambda x: x),
    # < 16G: 4G
    (16 * GB, lambda x: 4 * GB),
    # < 64G: 1/2 M up to max
    (64 * GB, lambda x: x / 2),
]
```

The recommended swap sizing from Red Hat differs slightly:

  * **2GB or less of system RAM:** 2x RAM
  * **Over 2GB to 8GB:** 1x RAM
  * **Over 8GB to 64GB:** 4 GB minimum
  * **Over 64GB:** 4GB minimum

You can manually specify the swap memory (in bytes) with the `size:` parameter
in the cloud-config:

```yaml
swap:
    filename: /swapfile
    size: 2147483648  # 2 GiB
```

## Provisioning swap on an existing cloud instance

For existing instances, a swap file is often the easiest method for enabling
swap memory. In this example, we place a 2 GiB swap file in `/swapfile`:

```bash
# BTRFS only #################################################################
# We must disable copy-on-write updates for swap files on btrfs file systems.
# The 'swapon' step fails if you skip these steps.
truncate -s 0 /swapfile
chattr +C /swapfile
# BTRFS only #################################################################

# A 2 GiB swap file.
dd if=/dev/zero of=/swapfile bs=1MiB count=2048

# Set the correct permissions on the swap file.
chmod 0600 /swapfile

# Format the swapfile.
mkswap /swapfile

# Enable the swapfile.
swapon /swapfile

# Add it to /etc/fstab to enable it after reboot.
echo "/swapfile none swap defaults 0 0" >> /etc/fstab
```

Check to see if your swap memory is ready:

```console
$ cat /proc/swaps
Filename        Type    Size      Used    Priority
/swapfile       file    2097148   0       -2
```

## Conclusion

Swap memory provides two valuable benefits: a safety cushion when system RAM
usage increases to dangerous levels and a parking lot for rarely used memory
pages that are taking valuable space in the system RAM. That safety cushion
comes with a performance penalty since memory paging to disks is incredibly slow
relative to system memory.

Cloud deployments rarely see swap memory deployed, but swap still has benefits
there. Choosing to deploy swap allows for a safety cushion when applications
misbehave, but it could cause an application to perform slowly until the memory
consumption issues are resolved. Skipping swap removes the cushion but ensures
that a misbehaving application is immediately stopped. Cloud administrators must
have a plan in place to handle these situations.

Luckily, cloud-init appears on the vast majority of cloud instances and it
allows for swap file creation with only a few lines of YAML. Swap memory is also
easy to configure after the deployment via simple shell commands or scripts.

Whether you choose to deploy swap memory or to go without, ensure you have a
plan when the worst happens. Monitor systems appropriately and keep a holistic
view of the application that they serve.


[`mounts` module]: https://cloudinit.readthedocs.io/en/latest/topics/modules.html#mounts
[cloud-init's source code]: https://github.com/canonical/cloud-init/blob/fc5d541529d9f4a076998b7b4a3c90bb4be0000d/cloudinit/config/cc_mounts.py#L198-L209

Photo credit: [Matthew Waring via Unsplash](https://unsplash.com/photos/MJAoiige14E)