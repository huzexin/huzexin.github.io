---
layout: post
title: sqlite移植for symbian
tags:
- sqlite
- symbian
categories: sqlite
description: 
---
##背景
sqlite是个轻量级的数据库，本身在其它平台上有比较大的优势，目前项目中存贮逻辑比较多，wince的方案也是sqlite，看文档iOS系统已经兼容了。所以想在symbian上也用sqlite，保持方案的一致性。symbian系统不支持sqlite，需要自己移植。

##移植方案
整体的思路就是基于sqlite 3.0, 把文件读写的接口改成symbian的。有些涉及到dl的接口可以不用实现。实现基本的接口即可。因为本身symbian支持C语言，sqlite 本身也是C的。所以移植过程中，没有遇到兼容的问题。

**但是需要注意的是，别做成dll,只能做成static lib. dll里面不支持static，如果你非要做成dll,那需要把所有的static放到tls里面。我尝试过，逻辑比较多。而且改动较大。所以static lib就可以了**


##代码实现 

```c++
============================================================================

Name : sqlite_symbian.cpp
 
Author :
 
Version : 1.0
 
Copyright : Arthur Hu
 
Description : sqlite_symbian declaration
 
============================================================================
 
*/
 
#include "sqlite3.h"
#include "sqliteInt.h"
#include "os_common.h"
#include <f32file.h>
#include <stdio.h>
#include <string.h>
#include <bautils.h>
#include <utf.h>



void Utf8ToUnicode(const char* src, TDes& aDes)

{
  const TUint8* ptr = (const TUint8*) src;
  TPtrC8 srcPtr(ptr, User::StringLength(ptr));
  CnvUtfConverter::ConvertToUnicodeFromUtf8(aDes, srcPtr);
}


typedef struct SymbianFile SymbianFile;

struct SymbianFile
{
  const sqlite3_io_methods *pMethod; /*** Must be first ***/
  sqlite3_vfs *pVfs; /* The VFS used to open this file */
  RFs iFs;
  RFile iFile; 
  unsigned char locktype; /* Type of lock currently held on this file */
  short sharedLockByte; /* Randomly chosen byte used as a shared lock */
  char zPath[255]; /* Full pathname of this file */
};



int FileClose(sqlite3_file* f)
{
  SymbianFile* file = (SymbianFile*) f;
  file->iFile.Close();
  file->iFs.Close();
  return SQLITE_OK;
}

int FileRead(sqlite3_file* f, void* data, int len, sqlite3_int64 offset)
{
  SymbianFile* file = (SymbianFile*) f;
  TInt off = offset;
  TInt err = file->iFile.Seek(ESeekStart, off);
  if (err != KErrNone)
  {
  return SQLITE_IOERR_READ;
  } 
  TPtr8 ptr((TUint8*) data, len);
  err = file->iFile.Read(ptr);
  if (err != KErrNone)
  {
  return SQLITE_IOERR_READ;
  }
  if (ptr.Length() < len)
  {
  return SQLITE_IOERR_SHORT_READ;
  }
  return SQLITE_OK;

}



int FileWrite(sqlite3_file* f, const void* data, int len, sqlite3_int64 offset)

{

  SymbianFile* file = (SymbianFile*) f;
  TInt off = offset;
  TInt err = file->iFile.Seek(ESeekStart, off);

  if (err != KErrNone)
  {
    return SQLITE_IOERR_WRITE;
  }

  TPtrC8 ptr((const TUint8*) data, len);
  err = file->iFile.Write(ptr);
  if (err != KErrNone)
  {
    return SQLITE_IOERR_WRITE;
  }
  return SQLITE_OK;

}

int FileTruncate(sqlite3_file* f, sqlite3_int64 size)

{

  SymbianFile* file = (SymbianFile*) f;

  return SQLITE_OK;

}

int FileSync(sqlite3_file* f, int flags)

{

  SymbianFile* file = (SymbianFile*) f;

  return SQLITE_OK;

}

int FileFileSize(sqlite3_file* f, sqlite3_int64 *pSize)

{

  SymbianFile* file = (SymbianFile*) f;

  TInt size = 0;

  TInt err = file->iFile.Size(size);

  if (err != KErrNone)

  {

    return SQLITE_OK;

  }

  *pSize = size;

  return SQLITE_OK;

}

int FileLock(sqlite3_file* f, int)

{

  SymbianFile* file = (SymbianFile*) f;

  return SQLITE_OK;

}

int FileUnlock(sqlite3_file* f, int)

{

  SymbianFile* file = (SymbianFile*) f;

  return SQLITE_OK;

}

int FileCheckReservedLock(sqlite3_file* f, int *pResOut)

{

  SymbianFile* file = (SymbianFile*) f;

  return SQLITE_OK;

}

int FileFileControl(sqlite3_file* f, int op, void *pArg)

{

  SymbianFile* file = (SymbianFile*) f;

  return SQLITE_OK;

}

int FileSectorSize(sqlite3_file* f)

{

  SymbianFile* file = (SymbianFile*) f;

  return SQLITE_OK;

}

int FileDeviceCharacteristics(sqlite3_file* f)

{

  SymbianFile* file = (SymbianFile*) f;

  return SQLITE_OK;

}

/* Methods above are valid for version 1 */

int FileShmMap(sqlite3_file* f, int iPg, int pgsz, int, void volatile**)

{

  SymbianFile* file = (SymbianFile*) f;

  return SQLITE_OK;

}

int FileShmLock(sqlite3_file* f, int offset, int n, int flags)

{

  SymbianFile* file = (SymbianFile*) f;

  return SQLITE_OK;

}

void FileShmBarrier(sqlite3_file* f)

{

  SymbianFile* file = (SymbianFile*) f;

}

int FileShmUnmap(sqlite3_file* f, int deleteFlag)

{

  SymbianFile* file = (SymbianFile*) f;

  return SQLITE_OK;

}



static const sqlite3_io_methods Methord =

{ 100, /* iVersion */

  FileClose, /* xClose */

  FileRead, /* xRead */

  FileWrite, /* xWrite */

  FileTruncate, /* xTruncate */

  FileSync, /* xSync */

  FileFileSize, /* xFileSize */

  FileLock, /* xLock */

  FileUnlock, /* xUnlock */

  FileCheckReservedLock, /* xCheckReservedLock */

  FileFileControl, /* xFileControl */

  FileSectorSize, /* xSectorSize */

  FileDeviceCharacteristics, /* xDeviceCharacteristics */

  FileShmMap, /* xShmMap */

  FileShmLock, /* xShmLock */

  FileShmBarrier, /* xShmBarrier */

  FileShmUnmap /* xShmUnmap */

};



int SymbianOpen(sqlite3_vfs * vfs, const char *zName, sqlite3_file* f,

int flags, int *pOutFlags)

{

  SymbianFile* file = (SymbianFile*) f;
  memset(file, 0, sizeof(SymbianFile));
  file->pMethod = &Methord;
  file->pVfs = vfs;
  strcpy(file->zPath, zName);
  TInt err = file->iFs.Connect();
  if (err != KErrNone)
  {
    return SQLITE_IOERR;
  }
  TFileName name;
  Utf8ToUnicode(zName, name);

  //RFile open
  if (flags & SQLITE_OPEN_CREATE)
  {
    if (BaflUtils::FileExists(file->iFs, name))
    {
      err = file->iFile.Open(file->iFs, name, EFileWrite | EFileRead);
    }
    else
    {
      err = file->iFile.Replace(file->iFs, name, EFileWrite | EFileRead);
    }
  }
  else
  {
    err = file->iFile.Open(file->iFs, name, EFileWrite | EFileRead);
  }

  if (err != KErrNone)
  {
    file->iFs.Close();
    return SQLITE_IOERR;
  }

  if (pOutFlags)
  {
    *pOutFlags = flags;
  }
  return SQLITE_OK;
}

int SymbianDelete(sqlite3_vfs*, const char *zName, int syncDir)
{

  RFs fs;
  TInt err = fs.Connect();
  if (err != KErrNone)
  {
    return SQLITE_IOERR;
  }
  TFileName name;
  Utf8ToUnicode(zName, name);
  fs.Delete(name);
  fs.Close();

  return SQLITE_OK;

}

int SymbianAccess(sqlite3_vfs*, const char *zName, int flags, int *pResOut)
{

  *pResOut = flags;
  return SQLITE_OK;
}

int SymbianFullPathname(sqlite3_vfs* pVfs, const char *zRelative, int nFull,
char *zFull)
{
  const char* temp = zRelative;
  temp = zFull;
  strcpy(zFull, zRelative);
  return SQLITE_OK;
}


void * SymbianDlOpen(sqlite3_vfs*, const char *zFilename)
{
  return NULL;
}

void SymbianDlError(sqlite3_vfs*, int nByte, char *zErrMsg)
{

}

void SymbianDlClose(sqlite3_vfs*, void*)
{

}

int SymbianRandomness(sqlite3_vfs*, int nByte, char *zOut)
{
  return SQLITE_NOMEM;
}

int SymbianSleep(sqlite3_vfs*, int microseconds)
{
  return SQLITE_NOMEM;
}

int SymbianCurrentTime(sqlite3_vfs*, double*)
{
  return SQLITE_NOMEM；
}

int SymbianGetLastError(sqlite3_vfs*, int, char *)
{
  return SQLITE_NOMEM;
}

/*

** The methods above are in version 1 of the sqlite_vfs object

** definition. Those that follow are added in version 2 or later

*/

int SymbianCurrentTimeInt64(sqlite3_vfs*, sqlite3_int64*)
{
  return 0;
}



SQLITE_API
int sqlite3_os_init(void)

{
  static sqlite3_vfs symbianVfs =

  { 1, /* iVersion */

    sizeof(SymbianFile), /* szOsFile */

    255, /* mxPathname */

    0, /* pNext */

    "symbian", /* zName */

    0, /* pAppData */



    SymbianOpen, /* xOpen */

    SymbianDelete, /* xDelete */

    SymbianAccess, /* xAccess */

    SymbianFullPathname, /* xFullPathname */

    SymbianDlOpen, /* xDlOpen */

    SymbianDlError, /* xDlError */

    NULL/*SymbianDlSym*/, /* xDlSym */

    SymbianDlClose, /* xDlClose */

    SymbianRandomness, /* xRandomness */

    SymbianSleep, /* xSleep */

    SymbianCurrentTime, /* xCurrentTime */

    SymbianGetLastError, /* xGetLastError */

  };

  sqlite3_vfs_register(&symbianVfs, 1);
  return SQLITE_OK;
}



SQLITE_API int sqlite3_os_end(void)
{
  return SQLITE_OK;
}
```

