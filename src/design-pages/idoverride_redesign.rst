#######################
ID-Override - Re-Design
#######################
Related Tickets
***************
* `Infopipe ListByCertificate does not return users if more than 1 override matches`_
* `[RFE] Need an option for sss_override so that uidnumber can be added locally even if it is not available on LDAP server`_
* `Unexpected interaction between overriding the primary group and the 'auto_private_groups = true' option`_
* `RFE: Permanently Persistent Local (on-system) SSS Overrides both automating importation (such as after the SSS DB is cleared and SSSD restarted) and a standard location for the override export files`_
* `IPA in trust. Client not seeing user with override set on AD user`_
* `User information is being removed from SSSD local Override database`_

.. _Infopipe ListByCertificate does not return users if more than 1 override matches: https://github.com/SSSD/sssd/issues/4456
.. _[RFE] Need an option for sss_override so that uidnumber can be added locally even if it is not available on LDAP server: https://bugzilla.redhat.com/show_bug.cgi?id=1896728
.. _Unexpected interaction between overriding the primary group and the 'auto_private_groups = true' option: https://bugzilla.redhat.com/show_bug.cgi?id=1921315
.. _RFE\: Permanently Persistent Local (on-system) SSS Overrides both automating importation (such as after the SSS DB is cleared and SSSD restarted) and a standard location for the override export files: https://bugzilla.redhat.com/show_bug.cgi?id=1939527
.. _IPA in trust. Client not seeing user with override set on AD user: https://bugzilla.redhat.com/show_bug.cgi?id=2010512
.. _User information is being removed from SSSD local Override database: https://bugzilla.redhat.com/show_bug.cgi?id=1452883


Problem Statement
*****************
The original design for id-overrides is described in
https://www.freeipa.org/page/V4/Migrating_existing_environments_to_Trust and
all the features described there should, of course, work as expected after the
redesign. Only the implementation should be updated with performance
and reliability in mind.

Initially the id-overrides were planned as a feature for the IPA provider. Some
time later, the local id-overrides were added keeping the way how id-overrides
were handled by the ipa provider. This meant that when the id-override was
added, the related user was looked up as well and added to the cache together
with the id-override data. If the user was removed from the cache for
whatever reasons, the matching id-override was removed from the cache as well.
This works well for IPA since there is a central storage for all current
id-overrides on the IPA servers. But the local id-override data is lost and
if the user is added to the cache again there will be no id-override data to
apply. So it would make sense to **decouple the id-override data in SSSD's cache
from the actual user and group objects**.

This decoupling would also **allow to handle the different change times** of
id-overrides, user and group objects more efficiently. While changes to groups
are expected to happen quite often by adding or removing members, changes to
user objects should be less frequent. ID-Overrides are typically created once
and will most probably never change. Currently, to look a user up, e.g. by name,
there will be first an LDAP search for an id-override with the given name.
Depending on whether an id-override was found, the user is looked up by the
id-override anchor or by name respectively. If no id-override was found before,
another search for the id-override is done with the anchor to see if other
attributes than the name is overwritten. For users or groups with no
id-overrides this means three LDAP searches.

The id-override data is typically stored in a separate tree/container in the
cache. There is an exception for the *"Default Trust View"* on IPA servers where
the id-override data is directly applied to the user or group objects in the
cache (see original design for the reason). With the new design it should be
possible to **get rid of this special handling**. *"Default Trust View"* will
become a normal view with no special handling executed by the front-end SSSD.
Instead of removing the whole id-override object automatically, only the
back-reference to the user or group object is removed. The id-override object
is deleted only if the source object is removed or the local id-override is
explicitly removed.

If the id-override data is downloaded once into the cache this can be handled
more efficiently because the id-override data is already present on the local
client. For local id-overrides it would allow to **bulk-load many id-overrides
into the cache** in a single run without looking up each corresponding user and
group. In addition, it would make it easier for other providers to implement
id-override support because relating the id-override and the original object
now happens inside the SSSD cache and does not involve a complicate sequence in
the lookup code of the backend.

There is currently a requirement that, when looking up attributes in
id-overrides, there should be only one result. While this is true for most
attributes, it is not true for e.g. searches by certificate. There are valid
use-cases where a single certificate from a Smartcard can be used to log into
different accounts. **This requirement must be removed**.

Finally, using id-overrides together with the domain option
*auto_private_groups* with its different values has to be carefully evaluated
and it should be clearly defined under which conditions a real group with a
GID matching the user's GID must exist and when it will be generated
automatically as user-private-group.

Solution Overview
*****************
Centralizing Object Retrieval in the Cache
==========================================
As stated above, when looking for a user or group object, up to three LDAP
operations may be required. Because of this, the complex code handling the
user or group lookups is repeated in several places requiring fixes and
modifications to be applied to all and each of them.

By moving the processing of these operations into the cache (not into sysdb) we
will allow the clients to execute a single operation and get the object with
the id-overrides already applied. The cache will internally execute all the
required operations (three if needed) and hide from the client all the details
about overriding. Overriding will be transparent for the clients. And any fix
or modification will have to be applied in a single place.

Bulk-Loading ID-Overrides into the Cache
========================================
While the id-override objects can be updated at runtime individually (although
we expect that they will rarely change at all) and new id-override object can
be added one after the other it might be necessary to load a large amount of
objects at startup if the cache is empty.

In the past we came across various issues with the files provider when loading
large /etc/passwd or /etc/group files. If we can't reliably load a larger
amount of objects in a reasonable startup time window the whole idea of just
loading all id-override objects won't work. Even then it make sense to
untangle the id-override lookups from the general user and group lookups in
the backend and provide dedicates request for id-override lookups.

But some basic tests indicate that at least in the order of 10000 objects the
time might be reasonable. As an example, I used objects like::

        dn: name=o1,cn=override,cn=sysdb
        objectClassXy: override
        uidNumber: 1000001
        gidNumber: 2000001
    
        dn: name=o2,cn=override,cn=sysdb
        objectClassXy: override
        uidNumber: 1000002
        gidNumber: 2000002
        ...
    
        # /bin/time ldbadd -H /var/lib/sss/db/cache_child.ad.vm.ldb /tmp/ldb.ldif 
        asq: Unable to register control with rootdse!
        Added 10000 records successfully
        0.29user 0.04system 0:00.37elapsed 93%CPU (0avgtext+0avgdata 35148maxresident)k
        0inputs+23880outputs (2985major+5125minor)pagefaults 0swaps


0.4s for writing 10000 objects seem reasonable to me. But we have to be
careful as a slight change will have different results. For instance, if all the
objects have the same gidNumber::

        dn: name=o1,cn=override,cn=sysdb
        objectClassXy: override
        uidNumber: 1000001
        gidNumber: 2000000
        
        dn: name=o2,cn=override,cn=sysdb
        objectClassXy: override
        uidNumber: 1000002
        gidNumber: 2000000
    
    
        # /bin/time ldbadd -H /var/lib/sss/db/cache_child.ad.vm.ldb /tmp/ldb.ldif 
        asq: Unable to register control with rootdse!
        Added 10000 records successfully
        4.18user 0.09system 0:04.34elapsed 98%CPU (0avgtext+0avgdata 32996maxresident)k
        0inputs+17192outputs (2245major+22254minor)pagefaults 0swaps


Or we use objectClass instead of objectClassXy::

        dn: name=o1,cn=override,cn=sysdb
        objectClass: override
        uidNumber: 1000001
        gidNumber: 2000000
        
        dn: name=o2,cn=override,cn=sysdb
        objectClass: override
        uidNumber: 1000002
        gidNumber: 2000000
    
        # /bin/time ldbadd -H /var/lib/sss/db/cache_child.ad.vm.ldb /tmp/ldb_noindex.ldif 
        asq: Unable to register control with rootdse!
        Added 10000 records successfully
        9.23user 0.22system 0:09.58elapsed 98%CPU (0avgtext+0avgdata 37580maxresident)k
        0inputs+17728outputs (2401major+58696minor)pagefaults 0swaps


In my opinion, the reason is the large index objects. Attributes uidNumber,
gidNumber and objectClass are indexed in SSSD's cache. In the first example
there is no objectClass attribute and uidNumber and gidNumber always have unique
values. So there will be a unique index object for each uidNumber and gidNumber
with a single entry.

In the second example, the gidNumber is the same for all users, the common
all-users-have-the-same-primary-group use-case. This means that the index
object for the GID 2000000 will become large and to add a new user with the
same gidNumber the object must be un-marshaled, it has to be checked if the
new object is already in the index and the index object must finally be
marshaled again::

        ldbsearch -H /var/lib/sss/db/cache_child.ad.vm.ldb -b '@INDEX:GIDNUMBER:2000000' -s base
        # record 1
        dn: @INDEX:GIDNUMBER:2000000
        @IDXVERSION: 2
        @IDX: name=o1,cn=override,cn=sysdb
        @IDX: name=o2,cn=override,cn=sysdb
        @IDX: name=o3,cn=override,cn=sysdb
        @IDX: name=o4,cn=override,cn=sysdb
        ...
        @IDX: name=o9998,cn=override,cn=sysdb
        @IDX: name=o9999,cn=override,cn=sysdb
        distinguishedName: @INDEX:GIDNUMBER:2000000
    
        # returned 1 records
        # 1 entries
        # 0 referrals
    
In the third example objectClass is indexed as well, leading to two large index
objects and, as a consequence, the third example runs about twice as long as the
second.

Since we cannot work without indexes, because every search would then have to
read and parse all objects in the cache, the attributes for the id-override
objects in the cache should be chosen to avoid large index objects.

Searches for user id-overrides are typically done by name, UID or certificate
while group id-overrides are searched by name or GID. A typical search filter
combines the objectClass with the searched attribute, e.g.
``(&(objectClass=override)(searchAttrName=searchValue))``. As seen above
objectClass should not be used and in SSSD there is already the un-indexed
objectCategory attribute which can be used to classify the object.

For user id-overrides it is expected that the name and the UID are unique, the
same is true for group id-overrides with respect to the GID. As seen above, the
attribute for the primary GID of the user does not have to be unique and it is
not an attribute we search for. Currently **we use the same attribute name
'gidNumber' for both but this should be avoided** to not bloat the index for the
group related attribute, which is expected to be unique, with the primary
groups of users. An attribute name like 'userGidNumber' should help to avoid
this.

Certificates are not expected to be unique in user id-overrides. This supports
use-cases where a single certificate (a single Smartcard) can be used to log
into different accounts. Think, e.g. of temporary/holiday replacements or
multiple accounts of a single physical user for different purposes. So we
cannot avoid indexes with multiple entries in this case. However it is
expected that for the above use-cases the multiple use of a single certificate
is limited to small numbers in the order of 10 where adding new entries to the
index should not cause a considerable slowdown. **Nevertheless it might be worth
to mention this in the documentation/man page.**

ID-override objects in the cache should be kept up-to-date using a scheme
similar to the one used for the sudo rules. **It should be configurable whether
to use full, smart or rule refresh.** For local id-overrides, the full refresh
would allow to bulk-load many id-overrides into the cache in a single run. This
can be done at start up. On the contrary, when there are too many id-overrides
on the server and only a few of them will be used on the host, the *rule refresh*
would prevent having those unneeded objects in the cache. The *smart refresh*
periodically updates id-override, user and group objects that are new or were
modified after the last update. Its primary goal is to keep the cache up-to-date
by fetching only small increments that do not generate large amounts of network
traffic. This configuration should be done for the backend in the domain section,
as it is the backend who will launch the bulk-load action and not the cache
itself.

Potential Optimizations
-----------------------
A potential additional optimization might be to use **indexed attributes with
names with a dedicated prefix**, e.g. 'ido' for id-override. With this, a search
filter for a user id-override with a given UID can be just ``idoUidNumber=UID``
and the object can be directly returned from the indexed search without
evaluating additional attributes. But since the additional evaluation would
just be a string comparison I would expect only a limited speedup, if any.

When looking for an attribute, for instance the UID, the 'ido' prefix would also
allow us to **look for the original user object or its id-override object in a
single operation**. The filter ``(|(idoUidNumber=UID)(uidNumber=UID))`` will
return the original object with that UID, or the id-override object producing
that UID. In the latter case, a second search will be necessary to lookup the
original object linked by the retrieved id-override object. In the former case,
a second lookup will also be necessary, this time to look-up an override
affecting other attributes which are not the UID. In any case, 2 operations
(instead of 3) will be necessary.

Managing ID-Overrides in SSSD's Cache
=====================================
Currently there are two types of sources for the id-override object. They are
either added locally with the ``sss_override`` command or read from LDAP. With
respect to LDAP, only IPA is currently supported properly, but with this
re-design adding support in plain LDAP or AD should be easier.

Local Overrides
---------------
Local overrides will be kept in the cache as it is currently done, in a
dedicated subtree, but they will also be kept in a persistent mode in a **local
file or database**. Because cache and configuration databases are sometimes
removed by the users, the local file/DB will be stored in a separate, persistent
directory. For instance, ``/var/spool/sssd``.

The ``sss_override`` command should just add or modify the id-override object
without trying to read the original object. This should speed adding
multiple id-overrides in a row up and would also allow to add the id-overrides
while the system is offline, e.g. when creating images or provisioning a
system in a special environment.

This separation also allows to keep the id-override in the cache while the
original object is deleted from the cache. The primary purpose here is to
avoid losing the id-override if the original object was removed from the
cache due to an error in SSSD (on the server side or in the configuration). The
drawback is that the id-override becomes useless if the original object was
removed on the serve side. But this issue is easier to handle than an
id-override that was lost accidentally. For instance, we could add a option to
``sss_override`` to look for local overrides which are missing the original
object.

The ``sss_override`` command currently writes the id-override objects directly
into the cache. It might be worth to consider implementing **S-Bus calls in the
backend** so that ``sss_override`` sends the data to the backend and the backend
writes them to minimize the number of processes writing into the cache. Backends,
especially IPA, must be able to reject local overrides based on some conditions,
for instance a configuration option whose default value is "accept." Besides
limiting the number of writer processes another advantage would be that the
backend is aware that an id-override was added. In case it is the first one, the
backend can switch on the id-override processing while, if the last id-override
object was removed, it can switch it off to save some cycles.

ID-Overrides from LDAP
-----------------------
During startup, if the cache is empty, all overrides should be read from the
server and added to the cache (see "Bulk-loading id-overrides into the
cache") in a dedicated subtree, separated from the local overrides. If the cache
already has id-override objects stored it has to be determined if it would be
easier to re-new all entries by loading everything from the server or if it
would be easier to just check for updates.

Similar to sudo rules, there would be a full refresh and a smart refresh.
But for id-overrides there will be a link to the original object in the cache
as well. It has to be checked during the implementation if it would be better
to

- only update the changed attributes during a refresh
- save the link attribute before the id-override object is renewed by
  removing the existing and adding the freshly read one during a refresh
- just remove and add the id-override object during refresh and run the
  linking process later if needed (see next section) 

To avoid redundant remove/add cycles if the object has not changed on the
server side it might be worth to use the timestamp cache for the id-override
objects.

In the sudo case the full refresh is basically used to handle objects which
are removed on the server side. Maybe it would be more efficient to replace
the full refresh by a scheme where a unique attribute (e.g. (original) DN or
the anchor string) is read for all id-override object both from the server and
from the cache and strings which are only in the cache are removed on the
server and can be removed from the cache as well.

As already mentioned, for the local id-overrides, the refresh task should notify
the backend if id-overrides are present or not so that the backend knows if
id-override processing is needed or not.

The "Default Trust View" on IPA Servers
---------------------------------------
As mentioned above there is a special handling of the id-overrides in IPA
servers with the "Default Trust View". Instead to storing the id-override
objects in a separate tree, the changed data is directly applied into the
original object and the original value are stored in attributes with the same
name and a prefix. This made sense at the time the id-overrides where
implemented originally because at this time the cache request code was not
available and each responder had individual code for looking up users and
group from the cache. With this it would have been cumbersome and error-prone
to handle the "Default Trust View" specially in every responder and as a
result the override attributes were added directly to the original cached AD
objects so that the responders could handle them as the objects coming from
IPA as if there were no id-overrides.

But nowadays with the common cache request code in the responders it would
make the code easier and would allow to remove the special handling of the
"Default Trust View" in IPA server mode if it would just be treated like any
other id-override with the id-override data in a separate tree without
applying the data directly. It would also help to avoid issues related to
lookups of user private groups of user from a trusted AD domain where e.g.
only the UID is overwritten. Since currently the UID is applied to the user
object in the cache the logic to lookup user-private groups breaks.

Since the backend now knows if id-overrides are defined or not it can signal
the frontends if override specific lookups can be skipped or not. 

Linking ID-Overrides and the Original Objects in SSSD's Cache
=============================================================
Since all id-override objects will be present in the cache in a known format,
the processing of the lookups can be handled in the cache itself. Freeing the
backends from dealing with overrides.

If there is a lookup by an attribute covered by id-overrides (name, POSIX ID,
certificate), we can check the id-overrides in case they are present. If an
id-override is found without a link to the original object in the cache,
it will use the anchor attribute to start a request in the id provider to
lookup the original object and if the request is successful the id-override
and the original object will be linked.

If no id-override is found, the request is forwarded to the id-provider and if
the new or updated object has no link to an id-override it will be checked if
there is an id-override for the object and either a link to the id-override
object or a self-link is created (self-linking means that overrides were
evaluated but no override was found -- no linking means that overrides were not
evaluated).

With the steps above it should be sufficient to send the request to the
backend if the cache request code comes across an id-override or original
object with a missing link attribute.

Extending ID-Mapping for AD with AD's POSIX Attributes (Long Term)
==================================================================
With the new scheme in place it would be possible, with a reasonable effort,
to extend the way UIDs and GIDs are generated for AD users. Currently, either
for all AD users and groups the UIDs and GIDs are calculated based on the SID
of the related object in AD, or POSIX UIDs and GIDs stored in the AD user and
group objects are used. In the latter case only the objects that have the
POSIX IDs set are visible on the Linux side. This might be useful and expected
if only a sub-set of the users and groups from AD should be visible.

But for existing environments where some users and groups have the UIDs and
GIDs already set (and should keep them) but now all users and groups from AD
should be visible on the Linux side, there is currently no good solution.
Either all users and groups will get new UIDs and GIDs, possibly breaking
existing applications. Or all users and groups must get POSIX attributes in
AD which will cause extra administrative work.

A solution would be to automatically create the POSIX IDs for all users and
groups based on the SID and use the data from the POSIX attributes as ID
overrides. With the new design in place the AD provider just needs a task to
refresh the id-override data, similar to the IPA provider's task, but with
different search filters.

Implementation Steps
====================
Given the critical nature, especially during startup with an empty cache, it
would be good to start with implementing a task which does read all
id-override data from the server side and write it to the cache as fast as
possible.

The next steps, which might be done in parallel is adding the linking
functionality to the common backend code and removing the id-override specific
code paths in the user and group lookup code from the IPA provider. Updating
the sss_override utility and integrating it in sssctl looks like another
task which can be handle separately.

Finally it should be checked if the cache request code can already handle the
new setup or if some modifications are needed here as well.

Authors
*******
* Sumit Bose <sbose@redhat.com>
* Alejandro López <allopez@redhat.com>
