ext4_setattr
========================================

Comments
----------------------------------------

path: fs/ext4/inode.c
```
/*
 * ext4_setattr()
 *
 * Called from notify_change.
 *
 * We want to trap VFS attempts to truncate the file as soon as
 * possible.  In particular, we want to make sure that when the VFS
 * shrinks i_size, we put the inode on the orphan list and modify
 * i_disksize immediately, so that during the subsequent flushing of
 * dirty pages and freeing of disk blocks, we can guarantee that any
 * commit will leave the blocks being flushed in an unused state on
 * disk.  (On recovery, the inode will get truncated and the blocks will
 * be freed, so we have a strong guarantee that no future commit will
 * leave these blocks visible to the user.)
 *
 * Another thing we have to assure is that if we are in ordered mode
 * and inode is still attached to the committing transaction, we must
 * we start writeout of all the dirty pages which are being truncated.
 * This way we are sure that all the data written in the previous
 * transaction are already on disk (truncate waits for pages under
 * writeback).
 *
 * Called with inode->i_mutex down.
 */
```

Arguments
----------------------------------------

```
int ext4_setattr(struct dentry *dentry, struct iattr *attr)
{
    struct inode *inode = d_inode(dentry);
    int error, rc = 0;
    int orphan = 0;
    const unsigned int ia_valid = attr->ia_valid;

    error = inode_change_ok(inode, attr);
    if (error)
        return error;

    if (is_quota_modification(inode, attr)) {
        error = dquot_initialize(inode);
        if (error)
            return error;
    }
```

ATTR_UID vs ATTR_GID
----------------------------------------

```
    if ((ia_valid & ATTR_UID && !uid_eq(attr->ia_uid, inode->i_uid)) ||
        (ia_valid & ATTR_GID && !gid_eq(attr->ia_gid, inode->i_gid))) {
        handle_t *handle;

        /* (user+group)*(old+new) structure, inode write (sb,
         * inode block, ? - but truncate inode update has it) */
        handle = ext4_journal_start(inode, EXT4_HT_QUOTA,
            (EXT4_MAXQUOTAS_INIT_BLOCKS(inode->i_sb) +
             EXT4_MAXQUOTAS_DEL_BLOCKS(inode->i_sb)) + 3);
        if (IS_ERR(handle)) {
            error = PTR_ERR(handle);
            goto err_out;
        }
        error = dquot_transfer(inode, attr);
        if (error) {
            ext4_journal_stop(handle);
            return error;
        }
        /* Update corresponding info in inode so that everything is in
         * one transaction */
        if (attr->ia_valid & ATTR_UID)
            inode->i_uid = attr->ia_uid;
        if (attr->ia_valid & ATTR_GID)
            inode->i_gid = attr->ia_gid;
        error = ext4_mark_inode_dirty(handle, inode);
        ext4_journal_stop(handle);
    }
```

ATTR_SIZE
----------------------------------------

```
    if (attr->ia_valid & ATTR_SIZE) {
        handle_t *handle;
        loff_t oldsize = inode->i_size;
        int shrink = (attr->ia_size <= inode->i_size);

        if (!(ext4_test_inode_flag(inode, EXT4_INODE_EXTENTS))) {
            struct ext4_sb_info *sbi = EXT4_SB(inode->i_sb);

            if (attr->ia_size > sbi->s_bitmap_maxbytes)
                return -EFBIG;
        }
        if (!S_ISREG(inode->i_mode))
            return -EINVAL;

        if (IS_I_VERSION(inode) && attr->ia_size != inode->i_size)
            inode_inc_iversion(inode);
```

### ext4_begin_ordered_truncate

```
        if (ext4_should_order_data(inode) &&
            (attr->ia_size < inode->i_size)) {
            error = ext4_begin_ordered_truncate(inode,
                                attr->ia_size);
            if (error)
                goto err_out;
        }
```

#### ext4_should_order_data

https://github.com/novelinux/linux-4.x.y/tree/master/fs/ext4/ext4_jbd2.h/ext4_should_order_data.md

#### ext4_begin_ordered_truncate

https://github.com/novelinux/linux-4.x.y/tree/master/fs/ext4/inode.c/ext4_begin_ordered_truncate.md

### ext4_journal_start

```
        if (attr->ia_size != inode->i_size) {
            handle = ext4_journal_start(inode, EXT4_HT_INODE, 3);
            if (IS_ERR(handle)) {
                error = PTR_ERR(handle);
                goto err_out;
            }
            if (ext4_handle_valid(handle) && shrink) {
                error = ext4_orphan_add(handle, inode);
                orphan = 1;
            }
            /*
             * Update c/mtime on truncate up, ext4_truncate() will
             * update c/mtime in shrink case below
             */
            if (!shrink) {
                inode->i_mtime = ext4_current_time(inode);
                inode->i_ctime = inode->i_mtime;
            }
            down_write(&EXT4_I(inode)->i_data_sem);
            EXT4_I(inode)->i_disksize = attr->ia_size;
```

https://github.com/novelinux/linux-4.x.y/tree/master/fs/ext4/ext4_jbd2.h/ext4_journal_start.md

### ext4_mark_inode_dirty

```
            rc = ext4_mark_inode_dirty(handle, inode);
            if (!error)
                error = rc;
            /*
             * We have to update i_size under i_data_sem together
             * with i_disksize to avoid races with writeback code
             * running ext4_wb_update_i_disksize().
             */
            if (!error)
                i_size_write(inode, attr->ia_size);
            up_write(&EXT4_I(inode)->i_data_sem);
            ext4_journal_stop(handle);
            if (error) {
                if (orphan)
                    ext4_orphan_del(NULL, inode);
                goto err_out;
            }
        }
```

```
        if (!shrink)
            pagecache_isize_extended(inode, oldsize, inode->i_size);

        /*
         * Blocks are going to be removed from the inode. Wait
         * for dio in flight.  Temporarily disable
         * dioread_nolock to prevent livelock.
         */
        if (orphan) {
            if (!ext4_should_journal_data(inode)) {
                ext4_inode_block_unlocked_dio(inode);
                inode_dio_wait(inode);
                ext4_inode_resume_unlocked_dio(inode);
            } else
                ext4_wait_for_tail_page_commit(inode);
        }
        /*
         * Truncate pagecache after we've waited for commit
         * in data=journal mode to make pages freeable.
         */
        truncate_pagecache(inode, inode->i_size);
        if (shrink)
            ext4_truncate(inode);
    }
```

mark_inode_dirty
----------------------------------------

```
    if (!rc) {
        setattr_copy(inode, attr);
        mark_inode_dirty(inode);
    }

    /*
     * If the call to ext4_truncate failed to get a transaction handle at
     * all, we need to clean up the in-core orphan list manually.
     */
    if (orphan && inode->i_nlink)
        ext4_orphan_del(NULL, inode);

    if (!rc && (ia_valid & ATTR_MODE))
        rc = posix_acl_chmod(inode, inode->i_mode);

err_out:
    ext4_std_error(inode->i_sb, error);
    if (!error)
        error = rc;
    return error;
}
```