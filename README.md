<div align="center">

## Read/write structures to/from a file


</div>

### Description

Generic structure <> disk file manipulations. These functions

form a basic template for reading and writing structures to a

sequential file. This template is probably most useful for files

with 500 or less records and eliminates the need for a more

elaborate file handler such as C-Tree, DB-Vista, Mix etc.

Routines to put data in the struct is out of scope here.

Written by Lynn Nash 8/28/91 and donated to the public domain.
 
### More Info
 


<span>             |<span>
---                |---
**Submitted On**   |
**By**             |[Bob Stout \(republished under Open Content License\)](https://github.com/Planet-Source-Code/PSCIndex/blob/master/ByAuthor/bob-stout-republished-under-open-content-license.md)
**Level**          |Beginner
**User Rating**    |4.0 (24 globes from 6 users)
**Compatibility**  |C, C\+\+ \(general\)
**Category**       |[Files](https://github.com/Planet-Source-Code/PSCIndex/blob/master/ByCategory/files__3-2.md)
**World**          |[C / C\+\+](https://github.com/Planet-Source-Code/PSCIndex/blob/master/ByWorld/c-c.md)
**Archive File**   |[](https://github.com/Planet-Source-Code/bob-stout-republished-under-open-content-license-read-write-structures-to-from-a-file__3-564/archive/master.zip)

### API Declarations

```
/* +++Date last modified: 05-Jul-1997 */
/*
** SNIPPETS header file for STRUCFIL.C
*/
#ifndef STRUCFIL__H
#define STRUCFIL__H
#include <stdio.h>
/* make sure the record structure is byte aligned */
#if defined(_MSC_VER) || defined(_QC) || defined(__WATCOMC__)
 #pragma pack(1)
#elif defined(__ZTC__)
 #pragma ZTC align 1
#elif defined(__TURBOC__) && (__TURBOC__ > 0x202)
 #pragma option -a-
#endif
static struct blackbook {
  int delete_flag;    /* 0 = active -1 = deleted */
  int recordnum;     /* a sequential number in the file */
  /* The data fields in asciiz. */
  char firstname[11];
  char lastname[16];
  char addr[26];
  char city[16];
  char state[3];
  char zip[10];
  char phone[11];
} rec, oldrec;       /* 97 byte record * 2 */
/* set structure alignment to default */
#if defined (_MSC_VER) || defined(_QC) || defined(__WATCOMC__)
 #pragma pack()
#elif defined (__ZTC__)
 #pragma ZTC align
#elif defined(__TURBOC__) && (__TURBOC__ > 0x202)
 #pragma option -a.
#endif
FILE * open_file(char *filename);
int  datadd(void);
int  data_delete(void);
int  data_read(long recnum);
int  data_update(void);
int  read_forward(void);
int  read_backward(void);
#endif /* STRUCFIL__H */
```


### Source Code

```
/* +++Date last modified: 05-Jul-1997 */
/*******************************************************************
 * Generic structure <> disk file manipulations. These functions
 * form a basic template for reading and writing structures to a
 * sequential file. This template is probably most useful for files
 * with 500 or less records and eliminates the need for a more
 * elaborate file handler such as C-Tree, DB-Vista, Mix etc.
 * Routines to put data in the struct is out of scope here.
 * Written by Lynn Nash 8/28/91 and donated to the public domain.
 */
#include <io.h>
#include <string.h>
#include <stdlib.h>
#include "sniptype.h"
#include "strucfil.h"
/*-------------------- general globals ---------------------*/
static long cur_rec = 0;   /* the current record number */
static FILE *fsptr = NULL;  /* fixed record data file pointer */
/* if file exists open in read/write mode else create file */
FILE * open_file(char *filename)
{
   if (access(filename, 0) == 0)
      fsptr = fopen(filename, "rb+");
   else fsptr = fopen(filename, "wb+");
   return fsptr;            /* return the file pointer */
}
/* add new records to the data file */
int datadd(void)
{
   if (fsptr)
   {
      if (fseek(fsptr, 0L, SEEK_END) != 0)
         return Error_; /* seek failure */
      rec.delete_flag = 0; /* active record tag */
      rec.recordnum = (int) (ftell(fsptr) /
         (long) sizeof(struct blackbook));
      if (fwrite(&rec, sizeof(struct blackbook), 1, fsptr) != 1)
      {
         return Error_; /* write error */
      }
      else
      {
         /* put your clean up code here */
         return Success_;
      }
   }
   return Error_;
}
/* tag the last record read in the file as deleted */
int data_delete(void)
{
   if (fsptr)
   {
      if (fseek(fsptr, (long) sizeof(struct blackbook) * -1L,
         SEEK_CUR) != 0)
      {
         return Error_;
      }
      rec.delete_flag = -1;  /* tag the record as deleted */
      if (fwrite(&rec, sizeof(struct blackbook), 1, fsptr) != 1)
         return Error_;
      else return Success_;
   }
   return Error_;
}
/* read a random structure. If successful the global cur_rec will
 * contain the number of the last record read & it can be compared
 * to the number in the struct as a double check (belt & suspenders)
 */
int data_read(long recnum)
{
   if (fseek(fsptr, (long) sizeof(struct blackbook) * recnum,
      SEEK_SET) != 0)
   {
      return Error_;
   }
   cur_rec = recnum; /* keep tabs on record pointer in global */
   /* now read the record into save struct*/
   if (fread(&oldrec, sizeof(struct blackbook), 1, fsptr) != 1)
   {
      return Error_;
   }
   else             /* copy save struct to edit struct */
   {
      memcpy(&rec, &oldrec, sizeof(struct blackbook));
      return Success_;
   }
}
/* rewrite the last read record back to disk */
int data_update(void)
{
   if (memcmp(&rec, &oldrec, sizeof(struct blackbook)) == 0)
      return True_; /* no update needed */
   /* back up one record before writing */
   if (fseek(fsptr, (long) sizeof(struct blackbook) * -1L,
      SEEK_CUR) != 0)
   {
      return Error_; /* seek error */
   }
   /* now write the record */
   if (fwrite(&rec, sizeof(struct blackbook), 1, fsptr) != 1)
      return Error_; /* write error */
   return Success_;
}
/* get the next valid record in the file */
int read_forward(void)
{
   do
   {
      cur_rec++; /* upcount the record number */
      if (data_read(cur_rec) != 0)
      {
         cur_rec--; /* decrement the record number */
         return Error_;
      }
   } while (oldrec.delete_flag != 0); /* record read was deleted */
   return Success_;
}
/* get the previous valid record in the file */
int read_backward(void)
{
   do
   {
      cur_rec--; /* decrement the record number */
      if (cur_rec >= 0)
      {
         if ( data_read(cur_rec) != 0 )
         {
            cur_rec++; /* increment the record number */
            return Error_;
         }
      }
   } while (oldrec.delete_flag != 0); /* record read was deleted */
   return Success_;
}
```

