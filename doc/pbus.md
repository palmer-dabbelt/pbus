# `pbus`: A Global Bus

This document describes `pbus`, a globally distributed,
cryptographically secure bus. `pbus` provides a set of logs, each of
which is available anywhere in the world.

## Global Logs

Each log is signified by a single public key, with the corresponding
keypair being known as the "root keypair".  The root keypair is the
ultimate authority for a log, there is no way to revoke it.  The root
keypair can only be used to sign messages that authorize or deauthorize
administrative keypairs, which themselves are used to control access to
the various log functions.  Administrative keypairs can be used to
authorize or revoke keypairs to perform the following functions:

* Appending to the log, which involves creating a new message and
  proposing it to be committed to the log.
* Committing to the log, which involves determining the order of a
  particular message in the log.
* Reading from the log, which really means decrypting messages in the
  log.  This is handled by encryption rather than authentication.
* Archiving the log, which means following the set of messages back to
  the beginning of the log.  Note that while it is possible to read
  messages without being able to archive them, that is probably a bad
  idea.

These message types are all handled in-band: in other words, the log is
an ordered set of messages of the types:

* Add or remove a set of permissions from a key.
* Change the current head of the log.

Note that there is no fundamental difference between the types of
keypairs, so while it's possible for a single keypair to be used in all
capacities that's generally considered a bad idea for security reasons.
Likewise, reusing keypairs is also possible -- this is expected to be
more frequent.

### Appending a Message to a Log

One important design consideration is that it must be possible to
construct and commit a new message without actually being able to read
the contents of a log.

### Committing a Message to a Log

Since administrative messages posted to a log must be agreed upon by
committers, it is possible to kill a log by having enough of the
committers permanently go offline that consensus can no longer be
reached.  In order to rectify this, all administrative actions in a
message take place _before_ the commit of that message.  As a result, it
is possible to cause consensus violations via administrative messages --
administrators just shouldn't do this, as they can do a lot of other
less subtle bad things anyway (like promoting writers that produce
invalid messages).

### Reading the Head of a Log

At any point in time, a log has one unique head: the entry that has most
recently been agreed upon by the consensus protocol running on the
commit nodes.  Writers and committers work together to ensure that it is
invariant that all readers are always capable of decrypting the head of
the log.

Every message in the log contains some number of parent messages, and
assuming readers have archival permissions they can follow the message
set

### Reading all Messages in the Log

In order to ensure that new readers can access the entire log history
when they are added to the log, whenever a new reader is appended to the
log they are given the key to all the previous messages in the log.  In
order to achieve this, there is always one additional reader keypair that
has access to the log known as the chained reader.

Whenever a new reader is added or removed from the log, a new chained
reader is added to the log.  The old chained reader's private key is
encrypted to any new reader's private key as well as the current chained
reader's private key.  This ensures that any new reader has access to
all the previous messages in the log, either directly via its private
key or indirectly via a chain of the old chained readers.

Whenever an administrator is added, the chained reader's private key is
encrypted to that administrator.  Whenever an administrator is removed,
the chained reader is cycled.  Since administrators can add additional
readers to the log, there is no security problem with giving
administrator keys read access to the entire log.

There is no way to revoke read access from a reader once it's been
given, so removing a reader's access simply prevents it from accessing
any new messages committed to the log.  We rely on log rotation to
eventually remove entries encrypted with the compromised key from
circulation.

### Log Rotation

In order to revoke a keypair's read access, the read key is first
rotated.  This ensures that all new data written to a log is
inaccessible to the revoked key.  Since it's impossible to enforce data
deletion, the only way to ensure that a log is eventually made
unavailable when a reader's keypair is compromised is to ensure that it
is economically unfeasible to continue storing that data for all time by
periodically rotating the log.

The simplest type of log rotation is simply re-encrypting every entry in
the log.

In order to facilitate log rotation, each log entry can have multiple
parents.  `pbus` trusts the writers to maintain the invariant that
exactly the same state can be reproduced from following any path in the
log, and readers are not expected to verify this.

## Program Execution

In addition to the global logs, `pbus` contains mechanisms for launching
programs that process these logs in order to produce new ones.

## `pbus` Libraries

Libraries are provided that treat `pbus` logs like other common
protocols.
