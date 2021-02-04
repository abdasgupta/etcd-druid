# Compaction for ETCD

## Current Problem
To ensure recoverability of ETCD, backups of the database are taken at regular interval.
Backups are of two types: Full Snapshots and Incremental Snapshots. 

**Full Snapshots:** Full snapshot is a snapshot of the complete database at given point in time.The size of the database keeps changing with time and typically the size is relatively large (measured in 100s of megabytes or even in gigabytes. For this reason, full snapshots are taken after some large intervals.

**Incremental Snapshots:** Incremental Snapshots are collection of commands on ETCD database, obtained through running WATCH API Call on ETCD. After some short intervals, all the commands that are accumulated through WATCH API Call are saved in a file and named as Incremental Snapshots.

### **Recovery from the Snapshots:**
**Recovery from Full Snapshots:** As the full snapshots are snapshots of the complete database, the whole database can recoved form a full snapshot in one go. ETCD provides API Call to restore the database from a full snapshot file.

**Recovery from Incremental Snapshots:** Delta snapshots are collection of retrospective ETCD commands. So, to restore from Incremental snapshot file, the commands from the file are needed to be applied sequentially on ETCD database through ETCD Put/Delete API calls. As it is heavily dependent on ETCD calls sequentially, restoring from Incremental Snapshot files can take long if there are numerous commands captured in Incremental Snapshot files.

Delta snapshots are applied on top of running ETCD database. So, if there is inconsistency between the state of database at the point of applying and the state of the database when the delta snapshot commands were captured, restoration will fail.

Currently, in Gardener setup, ETCD is restored from the last full snapshot and then the delta snapshots, which were captured after the last full snapshot.

The main problem with this is that the complete restoration time can be unacceptably long if the rate of change coming into the etcd database is quite high because there are large amount delta snapshots to be applied sequentially.
A secondary problem is that, though auto-compaction is enabled for etcd, the auto-compaction period it is not quick enough to compact all the changes from the incremental snapshots being re-applied during the relatively short period of time of restoration (as compared to the actual period of time when the incremental snapshots were accumulated). This may lead to the etcd pod (the backup-restore sidecar container, to be precise) to run out of memory and/or storage space even if it is sufficient for normal operations.

## Solution
**Compaction command:** To help with the problem mentioned earlier, our proposal is to introduce `compact` subcommand with `etcdbrctl`. On execution of `compact` command, ETCD data directory will be restored from the snapstore. A new ETCD process will be spawned parallely using the restored data directory. Then the new ETCD database will be compacted and defragmented using ETCD API calls. The compaction will strip off the ETCD database of old revisions as per the ETCD auto-compaction configuration. The defragmentation will free up the unused fragment memory space released after compaction. Then a full snapshot of the compacted database will be saved in snapstore which then can be used as the base snapshot during any subsequent restoration (or compaction).

**How the solution works:** The newly introduced compact command does not disturb the running ETCD while capturing the compacted snapshot. The command is designed to run potentially separately (from the main ETCD process/container/pod). ETCD Druid may take care of starting the newly introduced compact command as a separate job (scheduled periodically and also on-demand if required) based on some user defined parameter.

### **Points to take care while saving the compacted snapshot:**
As compacted snapshot and the existing periodic full snapshots are taken by different processes running in different pods but accessing same store to save the snapshots, some problems may arise:
1.	When uploading the compacted snapshot to the snapstore, there is the problem of how does the restorer know when to start using the newly compacted snapshot. This communication needs to be atomic.
2.	With a regular schedule for compaction that happens potentially separately from the main etcd pod, is there a need for regular scheduled full snapshots anymore?

#### **How to swap full snapshot with compacted snapshot atomically**

Currently, full snapshots and the subsequent delta snapshots are grouped under same prefix path in the snapstore. When a full snapshot is created, it is placed under a prefix/directory with the name comprising of timestamp. Then subsequent delta snapshots are also pushed into the same directory. Thus each prefix/directory contains a single full snapshot and the subsequent delta snapshots. So far, it is the job of ETCDBR to start main ETCD process and snapshotter process which takes full snapshot and delta snapshot periodically. But as per our proposal, compaction will be running as parallel process to main ETCD process and snapshotter process. So we can't reliably co-ordinate between the processes to achieve switching to the compacted snapshot as the base snapshot atomically.```

**Current Directory Structure**
```yaml
- Backup-192345
    - Full-Snapshot-192345
    - Incremental-Snapshot-192355
    - Incremental-Snapshot-192365
    - Incremental-Snapshot-192375
- Backup-192789
    - Full-Snapshot-192789
    - Incremental-Snapshot-192799
    - Incremental-Snapshot-192809
    - Incremental-Snapshot-192819
```

To solve the problem, proposal is:
1. ETCDBR will take the first full snapshot after it starts main ETCD Process and snapshotter process. But after taking the first full snapshot, snapshotter will only continue taking delta snapshots but no further full snapshots.
2. Flatten the directory structure of backup folder. Save all the full snapshots, delta snapshots and compacted snapshots under same directory/prefix. Also, name snapshots after revision number instead of timestamps. Restorer will restore from compacted snapshots that have highest revision number in the name and delta snapshots that have higher revision numbers in name than the comapcted full snapshot.

**Proposed Directory Structure**
```yaml
Backup :
    - Full-Snapshot-revision-1
    - Incremental-Snapshot-revision-6-1 (Format- Incremental-Snapshot-revision-<Revision Number After Last Operation>-<Revision Number Before First Operation>)
    - Incremental-Snapshot-revision-12-6
    - Incremental-Snapshot-revision-14-12
    - Incremental-Snapshot-revision-18-14
    - Full-Snapshot-revision-18 (Compacted)
    - Incremental-Snapshot-revision-22-18
    - Incremental-Snapshot-revision-27-22
    - Incremental-Snapshot-revision-32-27
    - Incremental-Snapshot-revision-38-32
    - Full-Snapshot-revision-38 (Compacted)
    - Incremental-Snapshot-revision-41-38
    - Incremental-Snapshot-revision-43-41
    - Incremental-Snapshot-revision-48-43
    - Incremental-Snapshot-revision-52-48
```

3. A meta-data file might need to be added in the Backup directory to mark which compacted (or full) snapshot is ready to be used as the base for restoration. Restorer would study the meta-data file and pickup the ready full snapshot from there and apply all the delta snapshots having revision number greater than that. So, with the backup directory as above, if the content of metadata file is like following:

* Full-Snapshot-revision-1
* Full-Snapshot-revision-18

Then, during restoration Full-Snapshot-revision-18, and all the delta snaphots from Incremental-Snapshot-revision-22-18 to Incremental-Snapshot-revision-52-48 will be applied. **Restorer would restore from latest ready full snapshot and all subsequent delta snapshots.** As Full-Snapshot-revision-38 is not present in the metadata file, it would not be considered as ready yet.

Another approach to make sure that restorer only picks up snapshots which are ready, full snapshots can be named with extension `.part` until the snapshot is ready. When the snapshot is ready, `.part` extension can be removed from it's name. Restorer would avoid any file having extension as `.part`.  

So, if Full-Snapshot-revision-38 is not ready in the above example, it can be named as `Full-Snapshot-revision-38.part`. Restorer will apply Full-Snapshot-revision-18 and all the delta snaphots from Incremental-Snapshot-revision-22-18 to Incremental-Snapshot-revision-52-48. **Restorer would restore from latest ready full snapshot and all subsequent delta snapshots.**

This option is preferred as it avoids introducing a metadata file. But it would work only if renaming to remove the `.part` extension (upon completion of upload) is an atomic operation in all the supported snapstores. This needs to be evaluated.

#### Backward Compatibility
1. **Restoration** : The changes to handle the newly proposed backup directory structure must be backward compatible with older structures at least for restoration because we need have to restore from backups in the older structure. This includes the support for restoring from a backup without a metadata file if that is used in the actual implementation.
2. **Backup** : For new snapshots (even on a backup containing the older structure), the new structure may be used. The new structure must be setup automatically including creating the base full snapshot (and also the creation of a metadata file if necessary).
3. **Garbage collection** : The existing functionality of garbage collection of snapshots (full and incremental) that fall out of the backup retention policy must be compatible with both old and new backup folder structure. I.e. the snapshots in the older backup structure must be retained in their own structure and the snapshots in the proposed backup structure should be retained in the proposed structure. Once all the snapshots in the older backup structure go out of the retention policy and are garbage collected, we can think of removing the support for older backup folder structure.  

**Note:** ETCD Backup Restore needs to take one full snapshot just after starting. Compactor will work only if there is any full snapshot already present in the store. It is not limitation but a design choice
