.. BTRFS troubleshooting related pages index

Troubleshooting pages
=====================

Correctness related, permanent

Error: parent transid verify error
----------------------------------

Reason: result of a failed internal consistency check of the filesystem's metadata.
Type: permanent

.. code-block::

   [ 4007.489730] BTRFS error (device vdb): parent transid verify failed on 30736384 wanted 10 found 8

The b-tree nodes are linked together, a block pointer in the parent node
contains target block offset and generation that last changed this block. The
block it points to then upon read verifies that the block address and the
generation matches. This check is done on all tree levels.

The number in **faled on 30736384** is the logical block number, **wanted 10**
is the expected generation number in the parent node, **found 8** is the one
found in the target block.  The number difference between the generation can
give a hint when the problem could have happened, in terms of transaction
commits.

Once the mismatched generations are stored on the device, it's permanent and
cannot be easily recovered, because of information loss. The recovery tool
``btrfs restore`` is able to ignore the errors and attempt to restore the data
but due to the inconsistency in the metadata the data need to be verified by the
user.

The root cause of the error cannot be easily determined, possible reasons are:

* logical bug: filesystem structures haven't been properly updated and stored
* misdirected write: the underlying storage does not store the data to the exact
  address as expected and overwrites some other block
* storage device (hardware or emulated) does not properly flush and persist data
  between transactions so they get mixed up
* lost write without proper error handling: writing the block worked as viewed
  on the filesystem layer, but there was a problem on the lower layers not
  propagated upwards

Error: No space left on device (ENOSPC)
---------------------------------------

Type: transient

Space handling on a COW filesystem is tricky, namely when it's in combination
with delayed allocation, dynamic chunk allocation and parallel data updates.
There are several reasons why the ENOSPC might get reported and there's not just
a single cause and solution. The space reservation algorithms try to fairly
assign the space, fall back to heuristics or block writes until enough data are
persisted and possibly making old copies available.

The most obvious way how to exhaust space is to create a file until the data
chunks are full:

.. code-block::

   $ df -h .
   Filesystem      Size  Used Avail Use% Mounted on
   /dev/sda        4.0G  3.6M  2.0G   1% /mnt/

   $ cat /dev/zero > file
   cat: write error: No space left on device

   $ df -h .
   Filesystem      Size  Used Avail Use% Mounted on
   /dev/sdc        4.0G  2.0G     0 100% /mnt/data250

   $ btrfs fi df .
   Data, single: total=1.98GiB, used=1.98GiB
   System, DUP: total=8.00MiB, used=16.00KiB
   Metadata, DUP: total=1.00GiB, used=2.22MiB
   GlobalReserve, single: total=3.25MiB, used=0.00B

The data chunks have been exhausted, so there's really no space left where to
write. The metadata chunks have space but that can't be used for that purpose.

Metadata space got exhausted
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Cannot track new data extents, no inline files, no reflinks, no xattrs.
Deletion still works.

Balance does not have enough workspace
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Relocation of block groups requires a temporary work space, ie. area on the
device that's available for the filesystem but without any other existing block
groups. Before balance starts a check is performed to verify the requested
action is possible. If not, ENOSPC is returned.

TODO
----

Transient

- enospc

- operation cannot be done

Possibly both

- checksum errors from changes on the medium under hands

- transient because of direct io

- stored from faulty data in memory
