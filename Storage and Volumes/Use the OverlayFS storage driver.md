# Use the OverlayFS storage driver
- `OverlayFS` is a modern union filesystem that is similar to AUFS, but faster and with a simpler implementation. 
- Docker provides two storage drivers for `OverlayFS`: the original `overlay`, and the newer and more stable `overlay2`.
- This topic refers to the Linux kernel driver as OverlayFS and to the Docker storage driver as overlay or overlay2.
# Prerequisites
OverlayFS is supported if you meet the following prerequisites:
- The overlay2 driver is supported on Docker Engine - Community, and Docker EE 17.06.02-ee5 and up, and is the recommended storage driver.
- Version 4.0 or higher of the Linux kernel, or RHEL or CentOS using version 3.10.0-514 of the kernel or higher. If you use an older kernel, you need to use the overlay driver, which is not recommended.
- The overlay and overlay2 drivers are supported on xfs backing filesystems, but only with d_type=true enabled.
# Configure Docker with the overlay or overlay2 storage driver
- To configure Docker to use the overlay storage driver your Docker host must be running version 3.18 of the Linux kernel (preferably newer) with the overlay kernel module loaded. 
- For the overlay2 driver, the version of your kernel must be 4.0 or newer.
The steps below outline how to configure the overlay2 storage driver. If you need to use the legacy overlay driver, specify it instead.
## Stop Docker
```
sudo systemctl stop docker
```
## Copy the contents of /var/lib/docker to a temporary location
```cp -au /var/lib/docker /var/lib/docker.bk```
## Edit /etc/docker/daemon.json
Add following 
```
{
  "storage-driver": "overlay2"
}
```
## Start Docker
```
sudo systemctl start docker
```
## Verify 
```
docker info
```
# How the overlay2 driver works?
- OverlayFS layers two directories on a single Linux host and presents them as a single directory. 
- These directories are called layers and the unification process is referred to as a union mount. 
- OverlayFS refers to the lower directory as lowerdir and the upper directory a upperdir
- The unified view is exposed through its own directory called merged.
- The overlay2 driver natively supports up to 128 lower OverlayFS layers. 
-  This capability provides better performance for layer-related Docker commands such as docker build and docker commit, and consumes fewer inodes on the backing filesystem.
# Image and container layers on-disk
- After downloading a five-layer image using docker pull ubuntu, you can see six directories under /var/lib/docker/overlay2.
```
ls -l /var/lib/docker/overlay2

total 24
drwx------ 5 root root 4096 Jun 20 07:36 223c2864175491657d238e2664251df13b63adb8d050924fd1bfcdb278b866f7
drwx------ 3 root root 4096 Jun 20 07:36 3a36935c9df35472229c57f4a27105a136f5e4dbef0f87905b2e506e494e348b
drwx------ 5 root root 4096 Jun 20 07:36 4e9fa83caff3e8f4cc83693fa407a4a9fac9573deaf481506c102d484dd1e6a1
drwx------ 5 root root 4096 Jun 20 07:36 e8876a226237217ec61c4baf238a32992291d059fdac95ed6303bdff3f59cff5
drwx------ 5 root root 4096 Jun 20 07:36 eca1e4e1694283e001f200a667bb3cb40853cf2d1b12c29feda7422fed78afed
drwx------ 2 root root 4096 Jun 20 07:36 l
```
- The new l (lowercase L) directory contains shortened layer identifiers as symbolic links. 
- The lowest layer contains a file called link, which contains the name of the shortened identifier, and a directory called diff which contains the layer’s contents.
-  It also contains a merged directory, which contains the unified contents of its parent layer and itself, and a work directory which is used internally by OverlayFS.
- To view the mounts which exist when you use the overlay storage driver with Docker, use the mount command.
``
 mount | grep overlay`
``
# How the overlay driver works?
- OverlayFS layers two directories on a single Linux host and presents them as a single directory. 
- These directories are called layers and the unification process is referred to as a union mount.
- OverlayFS refers to the lower directory as lowerdir and the upper directory a upperdir
- The unified view is exposed through its own directory called merged.
- The diagram below shows how a Docker image and a Docker container are layered.
![](https://docs.docker.com/storage/storagedriver/images/overlay_constructs.jpg)
- Where the image layer and the container layer contain the same files, the container layer “wins” and obscures the existence of the same files in the image layer.
- The overlay driver only works with two layers. This means that multi-layered images cannot be implemented as multiple OverlayFS layers. 
-  Instead, each image layer is implemented as its own directory under /var/lib/docker/overlay
- Hard links are then used as a space-efficient way to reference data shared with lower layers.
- The use of hardlinks causes an excessive use of inodes, which is a known limitation of the legacy overlay storage driver, and may require additional configuration of the backing filesystem. 
## Image and container layers on-disk
The following docker pull command shows a Docker host downloading a Docker image comprising five layers.
```
docker pull ubuntu

Using default tag: latest
latest: Pulling from library/ubuntu

5ba4f30e5bea: Pull complete
9d7d19c9dc56: Pull complete
ac6ad7efd0f9: Pull complete
e7491a747824: Pull complete
a3ed95caeb02: Pull complete
Digest: sha256:46fb5d001b88ad904c5c732b086b596b92cfb4a4840a3abd0e35dbb6870585e4
Status: Downloaded newer image for ubuntu:latest
```
## THE IMAGE LAYERS
- Each image layer has its own directory within /var/lib/docker/overlay/, which contains its contents, as shown below. The image layer IDs do not correspond to the directory IDs.
```
ls -l /var/lib/docker/overlay/
```
- The image layer directories contain the files unique to that layer as well as hard links to the data that is shared with lower layers. This allows for efficient use of disk space.
## THE CONTAINER LAYER
- Containers also exist on-disk in the Docker host’s filesystem under /var/lib/docker/overlay/
-  If you list a running container’s subdirectory using the ls -l command, three directories and one file exist:
```
ls -l /var/lib/docker/overlay/<directory-of-running-container>

total 16
-rw-r--r-- 1 root root   64 Jun 20 16:39 lower-id
drwxr-xr-x 1 root root 4096 Jun 20 16:39 merged
drwxr-xr-x 4 root root 4096 Jun 20 16:39 upper
drwx------ 3 root root 4096 Jun 20 16:39 work
```
- The lower-id file contains the ID of the top layer of the image the container is based on, which is the OverlayFS lowerdir.
- The upper directory contains the contents of the container’s read-write layer, which corresponds to the OverlayFS upperdir.
- The merged directory is the union mount of the lowerdir and upperdir, which comprises the view of the filesystem from within the running container.
- The work directory is internal to OverlayFS.
- To view the mounts which exist when you use the overlay storage driver with Docker, use the mount command. The output below is truncated for readability.
```
 mount | grep overlay
```
# How container reads and writes work with overlay or overlay2?
## Reading files
Consider three scenarios where a container opens a file for read access with overlay.
### The file does not exist in the container layer:
If a container opens a file for read access and the file does not already exist in the container (upperdir) it is read from the image (lowerdir). This incurs very little performance overhead.
### The file only exists in the container layer
 If a container opens a file for read access and the file exists in the container (upperdir) and not in the image (lowerdir), it is read directly from the container.
 ### The file exists in both the container layer and the image layer
 If a container opens a file for read access and the file exists in the image layer and the container layer, the file’s version in the container layer is read. Files in the container layer (upperdir) obscure files with the same name in the image layer (lowerdir).
 ## Modifying files or directories
 ### Writing to a file for the first time
 The first time a container writes to an existing file, that file does not exist in the container (upperdir). 
 - The overlay/overlay2 driver performs a copy_up operation to copy the file from the image (lowerdir) to the container (upperdir). 
 - The container then writes the changes to the new copy of the file in the container layer.
 - However, OverlayFS works at the file level rather than the block level. 
 - This means that all OverlayFS copy_up operations copy the entire file, even if the file is very large and only a small part of it is being modified.
 - This can have a noticeable impact on container write performance. However, two things are worth noting:
 1. The copy_up operation only occurs the first time a given file is written to. Subsequent writes to the same file operate against the copy of the file already copied up to the container.
 2. OverlayFS only works with two layers. This means that performance should be better than AUFS, which can suffer noticeable latencies when searching for files in images with many layers. This advantage applies to both overlay and overlay2 drivers. overlayfs2 is slightly less performant than overlayfs on initial read, because it must look through more layers, but it caches the results so this is only a small penalty.
 ### Deleting files and directories
 - When a file is deleted within a container, a `whiteout` file is created in the container (upperdir). The version of the file in the image layer (lowerdir) is not deleted (because the lowerdir is read-only). However, the `whiteout` file prevents it from being available to the container.
- When a directory is deleted within a container, an opaque directory is created within the container (upperdir). This works in the same way as a whiteout file and effectively prevents the directory from being accessed, even though it still exists in the image (lowerdir).
### Renaming directories
Calling rename(2) for a directory is allowed only when both the source and the destination path are on the top layer. Otherwise, it returns EXDEV error (“cross-device link not permitted”). Your application needs to be designed to handle EXDEV and fall back to a “copy and unlink” strategy.
# OverlayFS and Docker Performance
- Both overlay2 and overlay drivers are more performant than aufs and devicemapper
- In certain circumstances, overlay2 may perform better than btrfs as well. 
However, be aware of the following details.
## Page Caching
- OverlayFS supports page cache sharing. 
- Multiple containers accessing the same file share a single page cache entry for that file.
- This makes the overlay and overlay2 drivers efficient with memory and a good option for high-density use cases such as PaaS.
## copy_up
- As with AUFS, OverlayFS performs copy-up operations whenever a container writes to a file for the first time. 
- This can add latency into the write operation, especially for large files. 
- However, once the file has been copied up, all subsequent writes to that file occur in the upper layer, without the need for further copy-up operations.
- The OverlayFS copy_up operation is faster than the same operation with AUFS, because AUFS supports more layers than OverlayFS and it is possible to incur far larger latencies if searching through many AUFS layers.
- overlay2 supports multiple layers as well, but mitigates any performance hit with caching.
## Inode limits
- Use of the legacy overlay storage driver can cause excessive inode consumption. 
- This is especially true in the presence of a large number of images and containers on the Docker host. 
- The only way to increase the number of inodes available to a filesystem is to reformat it. 
-  To avoid running into this issue, it is highly recommended that you use overlay2 if at all possible.
# Performance best practices
## Use fast storage
## Use volumes for write-heavy workloads:
# Limitations on OverlayFS compatibility
To summarize the OverlayFS’s aspect which is incompatible with other filesystems:
## open(2)
OverlayFS only implements a subset of the POSIX standards. This can result in certain OverlayFS operations breaking POSIX standards. One such operation is the copy-up operation.
## rename(2)
OverlayFS does not fully support the rename(2) system call.




