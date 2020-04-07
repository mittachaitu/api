---
oep-number: 2
title: Pool Expansion When Underlying Disk Expanded
authors:
  - "@mittachaitu"
owners:
  - "@vishnuitta"
  - "@kmova"
  - "@prateekpandey14"
  - "@sonasingh46"
editor: "@mittachaitu"
creation-date: 2020-03-20
last-updated: 2020-04-07
status: Implementable
---

# Pool Expansion When Underlying Disk Expanded

## Table of Contents

- [Pool Expansion When Underlying Disk Expanded](#pool-expansion-when-underlying-disk-expanded)
	- [Table of Contents](#table-of-contents)
	- [Summary](#summary)
	- [Motivation](#motivation)
		- [Goals](#goals)
		- [Non-Goals](#non-goals)
	- [Proposal](#proposal)
		- [User Stories](#user-stories)
		- [Current Implementation](#current-implementation)
		- [Proposed Implementations](#proposed-implementations)
		- [Steps to perform user stories](#steps-to-perform-user-stories)
			- [Pool Expansion Steps:](#pool-expansion-steps)
		- [Low Level Design](#low-level-design)
			- [Work Flow](#work-flow)
	- [Drawbacks](#drawbacks)
	- [Alternatives](#alternatives)
	- [Infrastructure Needed](#infrastructure-needed)

## Summary

- This proposal includes design details of the pool expansion when underlying disks were expanded.

## Motivation

There were cases where OpenEBS users expand the pool by expanding underlying disks(which were used by pool) when space on the pool got exhausted.

### Goals

- Expand cStor pool when the underlying disk got expanded.

### Non-Goals

- If user dedicated the first part of partitioned disk for cStor and second part for third-party then pool expansion can't be performed   automatically untill and unless user extend the first partition.

## Proposal

### User Stories

As a OpenEBS user I should be able to expand the pool when underlying disk got expanded.

### Current Implementation

- When user creates the CSPC, CSPC controller which is watching for CSPC will get create request and creates cstorpoolinstances and cstorpooldeployments. CstorPoolInstance is responsible for holding the blockdevice information and other pool configurations where as cStor pool deployment have 3 containers cstor-pool-mgmt, cstor-pool and m-exporter as name conveyes cstor-pool container creates pool(ZFS file system) on top of the blockdevices, where as cstor-pool-mgmt sidecar container is responsible for managing these pool. If any changes required as of day 2 operations on the pool then pool configurations will be updated by adding BlockDevices, adding RaidGroup and replacing blockdevices cstor-pool-mgmt looks for any pending pool operations for CStorPoolInstance then performs pending operation on the pool.

Below are high level info to explain the Day2 operations that are currently supported.

- Pool Expansion When BlockDevice/raid group added:
	cstor-pool-mgmt will iterate over all the BlockDevices that exists in CStorPoolInstance resource and verify whether the corresponding BlockDevice/raid group is part of the pool. If that BlockDevice/raid group is not part of the pool then pool expansion will be triggered on newly added BlockDevice/raid group.

- BlockDevice Replacement In Pool:
    cstor-pool manager will iterate over all the BlockDevices present in CstorPoolInstance and verifies whether the current blockdevice is for BlockDevice replacement request[How cstor-pool-mgmt can know? We are maintaining annotation called openebs.io/predecessor: <replaced_blockdevice_name> on the claim of that blockdevice]? If requested BlockDevice is valid for replacement then the pool manager will perform BlockDevice replacement operation on the pool.

### Proposed Implementations

- CStor-pool Manger inbuilt CSPI controller watches for CSPI resources and tries to get the desired state via every sync operation. As part of reconciliation logic it will iterate over all the BlockDevices that exist on CStorPoolInstance resource if it found BlockDevice resource capacity(considerable changes) is greater than the existing capacity of BlockDevice on CstorPoolInstanceBlockDevice then we will follow below-mentioned steps to perform the [pool expansion](#Pool_Expansion_Steps).
- [What if cStor is consuming part of partitioned disk/entier partitioned disk? User has to extend the partition manualy after that NDM will update the capacity on the corresponding blockdevice CR. Once NDM updates the capacity on BlockDevice in next reconciliation of CSPI Pool Expansion will be triggered].
- [What if only one blockdevice got expanded in case of raid/mirror configuration? Pool expansion operation will be triggered but there won't be any change in size of the pool].

Currently CstorPoolInstanceBlockDevice holds below configurations 
```go
// CStorPoolInstanceBlockDevice contains the details of block devices that
// constitutes a raid group.
type CStorPoolInstanceBlockDevice struct {
	// BlockDeviceName is the name of the block device.
	BlockDeviceName string `json:"blockDeviceName"`

	// Capacity is the capacity of the block device.
	// It is system generated
	Capacity string `json:"capacity"`

	// DevLink is the dev link for block devices
	DevLink string `json:"devLink"`
}
```

### Steps to perform user stories

- User has to make sure that blockdevice capacity to needs to be updated.

#### Pool Expansion Steps:

Step1: Remove the buffering partition if exists which was created by cstor-pool container(ex: sdb9).

Step2: Expand the existing partition by executing following comamnd
       ```sh
       parted </disk> resizepart <partition_number> 100%
       ```

Step3: Inform the pool saying disk was expanded by triggering following command
       ```sh
       zpool online -e <pool_name> <disk_name>
       ```
Step4: If above steps are successfull then `Capacity` field in CStorPoolInstanceBlockDevice will be updated with latest size.
NOTE: Above steps automates the steps mentioned in [doc](https://github.com/openebs/openebs-docs/blob/day_2_ops/docs/resize-single-disk-pool.md).

### Low Level Design

#### Work Flow

- User/Admin will make sure the expanded size should reflect on corresponding blockdevice CR.
- During CSPI reconciliation cstor-pool-mgmt verifies whether the disk was expanded or not by comparing the capacity of CStorPoolInstanceBlockDevice with BlockDevice if there is considerable change in the capacity then cstor-pool-mgmt will trigger pool expansion steps.

NOTE: Pool will be expanded only if all the blockdevices in raidgroup were expanded in case of mirror/raid configuration. If it is stripe configuration expanding one blockdevice will expand the pool.

## Drawbacks

NA

## Alternatives

NA

## Infrastructure Needed

NA
