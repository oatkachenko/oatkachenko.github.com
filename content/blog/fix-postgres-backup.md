+++
title = "The Mystery of the Failing PostgreSQL Incremental Backups."
date = "2025-12-04T15:34:20+01:00"
draft = false
tags = ["postgres", "backups"]
+++

### Introduction

PostgreSQL's **incremental** backup feature, introduced to optimize backup storage and transfer times, promised to make a huge impact on how we handle large database backups. But what happens when your backup completes successfully, only to fail catastrophically during the restoration process? This is a story of how I uncovered a critical bug in PostgreSQL's codebase that silently broke backups of large, segmented relations.

### The First Encounter: When Success Becomes Failure

It started with a routine backup restoration test. Our production database, with several relatively small tables and one table exceeding 20Gb in size, had been successfully backed up using PostgreSQL's incremental backup tool. Once a week, on Mondays, we perform a full backup, and the rest of the week, incremental ones. The backup process completed without errors, the logs looked clean, and everything seemed perfect - until we tried to restore.
```
ERROR: file "base/16385/321697.1" has truncation block length 156789 in excess of segment size 131072
```
The error was cryptic. The backup had succeeded. The files were present. But when I run the `pg_combinebackup` tool to reconstruct the backup, it fails. The backup, which we depended on for disaster recovery, was essentially useless.
### Understanding the Components

Before diving into the bug, let's understand the key components involved:
##### Incremental Backups in PostgreSQL

Incremental backups work by comparing the current state of the database with a previous backup, transferring only the changed blocks(pages). This is achieved through **WAL summaries**.

##### WAL summaries

WAL (Write-Ahead Log) summaries contain metadata about what changed in the database:
 - **Relation identifier:** Which table/index was modified
 - **Modified block numbers:** Array (or bitmap) of blocks that changed
 - **Truncation block**: Set when a relation is truncated (reduced in size)
#### Relation Segmentation

PostgreSQL splits large relations into 1GB segments. Each segment can hold `RELSEG_SIZE` blocks (typically 131072 blocks at 8KB each = 1GB). A segmented table would have files like:
 - relation_name (segment 0, blocks 0-131071)
 - relation_name.1 (segment 1, blocks 131072-262143)
 - relation_name.2 (segment 1, blocks 262144-393215)
 - And so on...
#### The truncation block

When PostgreSQL's vacuum process removes pages (with all dead tuples) from the end of a heap file, it may truncate the relation, reducing its physical size. The truncation block number indicates where the relation now ends.

### The Investigation begins

##### Initial Hypothesis
At first, I suspected corruption in the backup files themselves, or perhaps network issues during transfer. But the backups were consistent, and the error was reproducible. It means the problem lies in PostgreSQL itself.

##### The Pattern Appears

After multiple attempts, I noticed:
 - Small tables (<1GB): Always restored successfully
 - Large tables (>1GB): Failed if they were modified (but not always)

So, I concentrated only on large relations.

##### Reading the Error Message

The error message held the key:
```
... truncation block length {x} in excess of segment size 131072
```
Why the hell is the restoration process processing a truncation block??? And what is the truncation block itself??? Without understanding this, I couldn't move forward, so I dived into code and started learning the things.

##### Diving Into the Code

##### The Backup Restoration Phase
First, I found a piece of code where the error arises:

```c
if (rf->truncation_block_length > RELSEG_SIZE)
    pg_fatal("file \"%s\" has truncation block length %u in excess of segment size %u",  filename, rf->truncation_block_length, RELSEG_SIZE);
```

##### The Backup Creation Phase

Then, I found where this property was calculated and set, 


```c
/* 
 * The truncation block length is the minimum length of the reconstructed 
 * file. Any block numbers below this threshold that are not present in
 * the backup need to be fetched from the prior backup. At or above this
 * threshold, blocks should only be included in the result if they are
 * present in the backup.
 */
 *truncation_block_length = size / BLCKSZ;
 if (BlockNumberIsValid(limit_block)) 
{ 
	unsigned relative_limit = limit_block - segno * RELSEG_SIZE;
	if (*truncation_block_length < relative_limit)
		*truncation_block_length = relative_limit; 
}
```

#### The Root Cause

The comparison operator was inverted. The code is intended to **cut** the relation at the truncation block length at the relative limit, not expand it. The logic should be:

"If the calculated truncation block length exceeds what's reasonable for this segment, cut it at the relative limit."

Instead, it was doing:

"If the calculated truncation block length is less than the relative limit, set it to the (potentially too-large) relative limit."

#### Why This Bug Was Hard to Catch

Several factors made this bug hard to catch:

1. **Small tables don't trigger it**: Testing with toy datasets wouldn't reveal the issue
2. **Backup succeeds silently**: No error during backup creation, only during restoration
3. **Requires specific conditions**: Tables must be >1GB **AND** vacuum must perform truncation
4. **Logical-sounding code**: The original code had comments and seemed reasonable at first glance
