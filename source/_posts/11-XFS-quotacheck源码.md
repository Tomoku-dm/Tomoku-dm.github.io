---
title: XFS-quotacheck源码
date: 2018-09-11 00:46:40
tags:
 - filesystem
 - kernel
---
### 流程图
```
graph TD
xfs_mountfs-->xfs_qm_mount_quotas
xfs_qm_mount_quotas-->XFS_QM_NEED_QUOTACHECK
XFS_QM_NEED_QUOTACHECK-->m_sb.sb_qflags
m_sb.sb_qflags-->xfs_qm_quotacheck
```
### xfs_mountfs
xfs_mount.c  会看 xfs 是否开启了 quota 调用 xfs_qm_mount_quotas（）
```
xfs_mountfs(
        struct xfs_mount        *mp)
{
        struct xfs_sb                *sbp = &(mp->m_sb);
        struct xfs_inode        *rip;
        __uint64_t                resblks;
        uint                        quotamount = 0;
        uint                        quotaflags = 0;
        int                        error = 0;

        xfs_sb_mount_common(mp, sbp);
        /*
         * Initialise the XFS quota management subsystem for this mount
         * xfs_qm_newmount 调用 quota mangement 去 mount 901行
         */
        if (XFS_IS_QUOTA_RUNNING(mp)) {
                error = xfs_qm_newmount(mp, &quotamount, &quotaflags);         /*****/
                if (error)
                        goto out_rtunmount;
        } else {
                ASSERT(!XFS_IS_QUOTA_ON(mp));

                /*
                 * If a file system had quotas running earlier, but decided to
                 * mount without -o uquota/pquota/gquota options, revoke the
                 * quotachecked license.
                 */
                if (mp->m_sb.sb_qflags & XFS_ALL_QUOTA_ACCT) {
                        xfs_notice(mp, "resetting quota flags");
                        error = xfs_mount_reset_sbqflags(mp);
                        if (error)
                                goto out_rtunmount;
                }
        }  
        /*
         * Complete the quota initialisation, post-log-replay component.
         */
        if (quotamount) {
                ASSERT(mp->m_qflags == 0);
                mp->m_qflags = quotaflags;

                xfs_qm_mount_quotas(mp);
        }

...    
}        
```

### xfs_qm_mount_quotas
根据 XFS_QM_NEED_QUOTACHECK(mp)的返回值决定是否做 `quotacheck`

```c
/*
 * This is called from xfs_mountfs to start quotas and initialize all
 * necessary data structures like quotainfo.  This is also responsible for
 * running a quotacheck as necessary.  We are guaranteed that the superblock
 * is consistently read in at this point.
 *
 * If we fail here, the mount will continue with quota turned off. We don't
 * need to inidicate success or failure at all.
 */
xfs_qm_mount_quotas(
        struct xfs_mount        *mp)
{
        int                        error = 0;
        uint                        sbf;

        /*
         * If quotas on realtime volumes is not supported, we disable
         * quotas immediately.
         */
        if (mp->m_sb.sb_rextents) {
                xfs_notice(mp, "Cannot turn on quotas for realtime filesystem");
                mp->m_qflags = 0;
                goto write_changes;
        }

        ASSERT(XFS_IS_QUOTA_RUNNING(mp));

        /*
         * Allocate the quotainfo structure inside the mount struct, and
         * create quotainode(s), and change/rev superblock if necessary.
         */
        error = xfs_qm_init_quotainfo(mp);
        if (error) {
                /*
                 * We must turn off quotas.
                 */
                ASSERT(mp->m_quotainfo == NULL);
                mp->m_qflags = 0;
                goto write_changes;
        }
        /*
         * If any of the quotas are not consistent, do a quotacheck.
         */
        if (XFS_QM_NEED_QUOTACHECK(mp)) {
                error = xfs_qm_quotacheck(mp);
                if (error) {
                        /* Quotacheck failed and disabled quotas. */
                        return;
                }
        }
        /*
         * If one type of quotas is off, then it will lose its
         * quotachecked status, since we won't be doing accounting for
         * that type anymore.
         */
        if (!XFS_IS_UQUOTA_ON(mp))
                mp->m_qflags &= ~XFS_UQUOTA_CHKD;
        if (!XFS_IS_GQUOTA_ON(mp))
                mp->m_qflags &= ~XFS_GQUOTA_CHKD;
        if (!XFS_IS_PQUOTA_ON(mp))
                mp->m_qflags &= ~XFS_PQUOTA_CHKD;
                
}                
```



#### XFS_IS_QUOTA_RUNNING(mp)  定义

```
//xfs_quota.h
#define XFS_QM_NEED_QUOTACHECK(mp) \
        ((XFS_IS_UQUOTA_ON(mp) && \
                (mp->m_sb.sb_qflags & XFS_UQUOTA_CHKD) == 0) || \
         (XFS_IS_GQUOTA_ON(mp) && \
                (mp->m_sb.sb_qflags & XFS_GQUOTA_CHKD) == 0) || \
         (XFS_IS_PQUOTA_ON(mp) && \
                (mp->m_sb.sb_qflags & XFS_PQUOTA_CHKD) == 0))
        
```
XFS_QM_NEED_QUOTACHECK(mp) 的返回值 是根据 mp->m_sb.sb_qflags 的 flag 。flag 来自 xfs_sb  的 *from   xfs 的 superblock。 

```
xfs_sb_quota_to_disk(
        struct xfs_dsb        *to,
        struct xfs_sb        *from)
{
        __uint16_t        qflags = from->sb_qflags;
__be16                sb_qflags;        /* quota flags */

```

### xfs_qm_quotacheck

最后调用的 quotacheck 函数
```
/*
 * Walk thru all the filesystem inodes and construct a consistent view
 * of the disk quota world. If the quotacheck fails, disable quotas.
 */
 
xfs_qm_quotacheck(
        xfs_mount_t        *mp)
{
        int                        done, count, error, error2;
        xfs_ino_t                lastino;
        size_t                        structsz;
        uint                        flags;
        LIST_HEAD                (buffer_list);
        struct xfs_inode        *uip = mp->m_quotainfo->qi_uquotaip;
        struct xfs_inode        *gip = mp->m_quotainfo->qi_gquotaip;
        struct xfs_inode        *pip = mp->m_quotainfo->qi_pquotaip;

        count = INT_MAX;
        structsz = 1;
        lastino = 0;
        flags = 0;

        ASSERT(uip || gip || pip);
        ASSERT(XFS_IS_QUOTA_RUNNING(mp));

        xfs_notice(mp, "Quotacheck needed: Please wait.");

        /*
         * First we go thru all the dquots on disk, USR and GRP/PRJ, and reset
         * their counters to zero. We need a clean slate.
         * We don't log our changes till later.
         */
        if (uip) {
                error = xfs_qm_dqiterate(mp, uip, XFS_QMOPT_UQUOTA,
                                         &buffer_list);
                if (error)
                        goto error_return;
                flags |= XFS_UQUOTA_CHKD;
        }

        if (gip) {
                error = xfs_qm_dqiterate(mp, gip, XFS_QMOPT_GQUOTA,
                                         &buffer_list);
                if (error)
                        goto error_return;
                flags |= XFS_GQUOTA_CHKD;
        }

        if (pip) {
                error = xfs_qm_dqiterate(mp, pip, XFS_QMOPT_PQUOTA,
                                         &buffer_list);
                if (error)
                        goto error_return;
                flags |= XFS_PQUOTA_CHKD;
        }

        do {
                /*
                 * Iterate thru all the inodes in the file system,
                 * adjusting the corresponding dquot counters in core.
                 */
                error = xfs_bulkstat(mp, &lastino, &count,
                                     xfs_qm_dqusage_adjust,
                                     structsz, NULL, &done);
                if (error)
                        break;

        } while (!done);

        /*
         * We've made all the changes that we need to make incore.  Flush them
         * down to disk buffers if everything was updated successfully.
         */
        if (XFS_IS_UQUOTA_ON(mp)) {
                error = xfs_qm_dquot_walk(mp, XFS_DQ_USER, xfs_qm_flush_one,
                                          &buffer_list);
        }
        if (XFS_IS_GQUOTA_ON(mp)) {
                error2 = xfs_qm_dquot_walk(mp, XFS_DQ_GROUP, xfs_qm_flush_one,
                                           &buffer_list);
                if (!error)
                        error = error2;
        }
        if (XFS_IS_PQUOTA_ON(mp)) {
                error2 = xfs_qm_dquot_walk(mp, XFS_DQ_PROJ, xfs_qm_flush_one,
                                           &buffer_list);
                if (!error)
                        error = error2;
        }

        error2 = xfs_buf_delwri_submit(&buffer_list);
        if (!error)
                error = error2;

        /*
         * We can get this error if we couldn't do a dquot allocation inside
         * xfs_qm_dqusage_adjust (via bulkstat). We don't care about the
         * dirty dquots that might be cached, we just want to get rid of them
         * and turn quotaoff. The dquots won't be attached to any of the inodes
         * at this point (because we intentionally didn't in dqget_noattach).
         */
        if (error) {
                xfs_qm_dqpurge_all(mp, XFS_QMOPT_QUOTALL);
                goto error_return;
        }

        /*
         * If one type of quotas is off, then it will lose its
         * quotachecked status, since we won't be doing accounting for
         * that type anymore.
         */
        mp->m_qflags &= ~XFS_ALL_QUOTA_CHKD;
        mp->m_qflags |= flags;
}         
```
