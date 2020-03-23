---
title: 关于chacl的分析
date: 2020-01-13 22:10:10
tags:
- acl
- NFS
categories:
- kernel
- nfs
---

## 关于chacl的分析
最近在nfs上面遇到chacl问题，这里记录下分析过程。

## ls命令
ls命令根据系统调用getxattr的返回值判断是否要打印"+"

https://github.com/coreutils/coreutils/blob/master/src/ls.c
```
			  
  if (! any_has_acl)
    modebuf[10] = '\0';
  else if (f->acl_type == ACL_T_LSM_CONTEXT_ONLY)
    modebuf[10] = '.';
  else if (f->acl_type == ACL_T_YES)
    modebuf[10] = '+'; // 有acl数据则打印"+"
```	

https://github.com/coreutils/gnulib/blob/master/lib/file-has-acl.c
```
int file_has_acl (char const *name, struct stat const *sb)
{
#if USE_ACL
  if (! S_ISLNK (sb->st_mode))
    {

# if GETXATTR_WITH_POSIX_ACLS

      ssize_t ret;

      ret = getxattr (name, XATTR_NAME_POSIX_ACL_ACCESS, NULL, 0); //是否有acl数据
      if (ret < 0 && errno == ENODATA)
        ret = 0;
      else if (ret > 0)
        return 1;

      if (ret == 0 && S_ISDIR (sb->st_mode))
        {
          ret = getxattr (name, XATTR_NAME_POSIX_ACL_DEFAULT, NULL, 0);
          if (ret < 0 && errno == ENODATA)
            ret = 0;
          else if (ret > 0)
            return 1;
        }

      if (ret < 0)
        return - acl_errno_valid (errno);
      return ret;
	...
```



## 3.10内核中getxattr

chacl会调用getxattr。

```
getxattr
  \- vfs_getxattr
         \- nfs3_getxattr
```

nfs3_getxattr@fs/nfs/nfs3acl.c
```
acl = nfs3_proc_getacl(inode, type);
if (IS_ERR(acl))
	return PTR_ERR(acl);
else if (acl) {
	if (type == ACL_TYPE_ACCESS && acl->a_count == 0)
		error = -ENODATA;
	else
		error = posix_acl_to_xattr(&init_user_ns, acl, buffer, size);
	posix_acl_release(acl);
} else
	error = -ENODATA;//如果nfs3_proc_getacl返回null则返回ENODATA
				
nfs3_proc_getacl@fs/nfs/nfs3acl.c

if (res.acl_access != NULL) {
		if (posix_acl_equiv_mode(res.acl_access, NULL) == 0) {//对于标准权限则置空acl
				posix_acl_release(res.acl_access);
				res.acl_access = NULL;
		}
}
```

nfs3_getxattr对于标准权限总是返回ENODATA，所以ls时不会出现"+"。


## 4.18内核getxattr
```
getxattr
 \- vfs_getxattr
       \- posix_acl_xattr_get
```
posix_acl_xattr_get@fs/posix_acl.c
```
acl = get_acl(inode, handler->flags);
if (IS_ERR(acl))
		return PTR_ERR(acl);
if (acl == NULL)
		return -ENODATA;
```
get_acl@fs/posix_acl.c
```
acl = get_cached_acl(inode, type);
if (!is_uncached_acl(acl))
		return acl;
```
				
如果inode->i_acl inode->i_default_acl有效则使用它,
否则执行nfs3_get_acl，nfs3_get_acl对于标准权限总是返回NULL，从而getxattr返回ENODATA.

nfs3_get_acl@fs/nfs/nfs3acl.c
```
if (res.acl_access != NULL) {
		if ((posix_acl_equiv_mode(res.acl_access, NULL) == 0) ||
			res.acl_access->a_count == 0) {
				posix_acl_release(res.acl_access);
				res.acl_access = NULL;
		}
}
```

在chacl之后如果inode->i_acl有效,getxattr返回acl数据。

## 结论
出现问题的根本原因是4.18内核使用generic acl功能框架，缓存了错误的acl，导致出问题。

最新的内核有[patch](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/fs/nfs/nfs3acl.c?h=v5.4&id=ded52fbe7020a5696b0b0a0fdbf234e37bf16f94)修正该问题。
即在setacl的时候总是置inode->i_acl无效。