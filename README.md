##vmbk

`vmbk` is a tool for backuping up KVM virtual machines. It relies heavily on [fi-backup.sh](https://github.com/dguerri/LibVirtKvm-scripts/blob/master/fi-backup.sh) for the actual interaction with libvirt and snapshotting.

At the time of writting, `vmbk` was known to work with `fi-backup.sh` version 2.1.0.

`vmbk` started as a simple wrapper script for `fi-backup.sh` so I could easily setup a backup cronjob. However `vmbk` now has a few extra features.

- - -

#### Extra Features
##### Relative Backing File Paths
The partial images produced by `fi-backup.sh` have a hardcoded backing file path. (This seems to be the default behavior of libvirt, rather than something specific to `fi-backup.sh`).

    # Example image with a hardcoded backing file path.
    ~$ qemu-img info example.bimg-20151031-031610 | grep "backing file:"
    backing file: /var/lib/libvirt/images/example.bimg-20151031-031610

`vmbk` can remove these hardcoded paths, in favor of relative paths (actually it presumes all parts of the chain are in the same directory):

    # After running `strip_backing_path` the backing file is now in a relative location.
    ~$ vmbk strip_backing_path
    ~$ qemu-img info example2.bimg-20151028-031722 | grep "backing file:"
    backing file: example2.bimg-20151028-031722 (actual path: /mnt/nfs/backups/vm-images/example2.bimg-20151028-031722

##### Chain validation
Moving partial images around and editing their metadata makes me nervous. Therefore `vmbk` can also validate the backing chain of images to ensure that every file required for restoration is readable. This feature takes advantage of how the `--backing-chain` flag of `qemu-img info` will do the interation for you.

    ~$ vmbk validate_chain

By default, whenever a backup is taken the chain is also validated.

- - -

###Example Usage
First use `vmbk.cfg.sample` to create your own `vmbk.cfg` file (currently this needs to be in the same directory `vmbk` itself).

Running the `do_backup` command does three things:
 * Firstly it interates through each VM, using `fi-backup.sh` to snapshot the disk and copy it to the backup directory.
 * It then runs `strip_backing_path` to remove the hardcoded chain paths for the partial images (this can be skipped by setting `CONVERT=0` in `vmbk.cfg`.
 * Finally it runs `validate_chain` to ensure qemu-img can read every file in the chain.


    ~$ vmbk do_backup

I suggest reading about how `fi-backup.sh`'s backup [process works](https://github.com/dguerri/LibVirtKvm-scripts/blob/master/NUTSNBOLTS.md) (as it's a little nuanced). However once a week I consolidate snapshots, and start the process again. This could also be considered doing a "full" backup.

    ~$ vmbk consolidate && vmbk do_backup

####Example cronjob:

In practise this is all done automaticaly with a simple cronjob:

    ~$ cat /etc/cron.d/backup-vms
    # 2am Sunday do "full" backup.
    0 2 * * 0 root /usr/local/bin/vmbk consolidate && /usr/local/bin/vmbk do_backup

    # 3.15am Mon-Sat do a incremental backup.
    15 3 * * 1-6 root /usr/local/bin/vmbk do_backup
