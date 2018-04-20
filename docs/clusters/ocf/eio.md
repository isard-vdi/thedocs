```bash
#!/bin/sh
#
#
#   EIO OCF RA. Does nothing but wait a few seconds, can be
#   configured to fail occassionally.
#
# Copyright (c) 2004 SUSE LINUX AG, Lars Marowsky-Br�e
#                    All Rights Reserved.
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of version 2 of the GNU General Public License as
# published by the Free Software Foundation.
#
# This program is distributed in the hope that it would be useful, but
# WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
#
# Further, this software is distributed without any warranty that it is
# free of the rightful claim of any third person regarding infringement
# or the like.  Any license provided herein, whether implied or
# otherwise, applies only to this software file.  Patent licenses, if
# any, provided herein do not apply to combinations of this program with
# other software, or any other product whatsoever.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write the Free Software Foundation,
# Inc., 59 Temple Place - Suite 330, Boston MA 02111-1307, USA.
#

#######################################################################
# Initialization:

: ${OCF_FUNCTIONS=${OCF_ROOT}/resource.d/heartbeat/.ocf-shellfuncs}
. ${OCF_FUNCTIONS}
: ${__OCF_ACTION=$1}

#######################################################################

meta_data() {
    cat <<END
<?xml version="1.0"?>
<!DOCTYPE resource-agent SYSTEM "ra-api-1.dtd">
<resource-agent name="eio4" version="1.0">
<version>1.0</version>

<longdesc lang="en">
EnhanceIO
</longdesc>
<shortdesc lang="en">EnhanceIO</shortdesc>

<parameters>
<parameter name="state" unique="1">
<longdesc lang="en">
Location to store the resource state in.
</longdesc>
<shortdesc lang="en">State file</shortdesc>
<content type="string" default="${HA_VARRUN}/EIO-{OCF_RESOURCE_INSTANCE}.state" />
</parameter>

<parameter name="passwd" unique="1">
<longdesc lang="en">
Fake password field
</longdesc>
<shortdesc lang="en">Password</shortdesc>
<content type="string" default="" />
</parameter>

<parameter name="fake" unique="0">
<longdesc lang="en">
Fake attribute that can be changed to cause a reload
</longdesc>
<shortdesc lang="en">Fake attribute that can be changed to cause a reload</shortdesc>
<content type="string" default="eio" />
</parameter>

<parameter name="op_sleep" unique="1">
<longdesc lang="en">
Number of seconds to sleep during operations.  This can be used to test how
the cluster reacts to operation timeouts.
</longdesc>
<shortdesc lang="en">Operation sleep duration in seconds.</shortdesc>
<content type="string" default="0" />
</parameter>

</parameters>

<actions>
<action name="start"        timeout="20" />
<action name="stop"         timeout="1800" />
<action name="monitor"      timeout="20" interval="10" depth="0"/>
<action name="reload"       timeout="20" />
<action name="migrate_to"   timeout="20" />
<action name="migrate_from" timeout="20" />
<action name="validate-all" timeout="20" />
<action name="meta-data"    timeout="5" />
</actions>
</resource-agent>
END
}

#######################################################################

# don't exit on TERM, to test that lrmd makes sure that we do exit
trap sigterm_handler TERM
sigterm_handler() {
    ocf_log info "They use TERM to bring us down. No such luck."
    return
}

eio_usage() {
    cat <<END
usage: $0 {start|stop|monitor|migrate_to|migrate_from|validate-all|meta-data}

Expects to have a fully populated OCF RA-compliant environment set.
END
}

eio_start() {
    /usr/local/bin/eio_drbd.sh start
    touch ${OCF_RESKEY_state}
    return $OCF_SUCCESS
}

eio_stop() {
    /usr/local/bin/eio_drbd.sh stop
    rm ${OCF_RESKEY_state}
    return $OCF_SUCCESS
}

eio_monitor() {
    # Monitor _MUST!_ differentiate correctly between running
    # (SUCCESS), failed (ERROR) or _cleanly_ stopped (NOT RUNNING).
    # That is THREE states, not just yes/no.

    if [ "$OCF_RESKEY_op_sleep" -ne "0" ]; then
        if [ -f ${VERIFY_SERIALIZED_FILE} ]; then
            # two monitor ops have occurred at the same time.
            # this is to verify a condition in the lrmd regression tests.
            ocf_log err "$VERIFY_SERIALIZED_FILE exists already"
            return $OCF_ERR_GENERIC
        fi

        touch ${VERIFY_SERIALIZED_FILE}
        sleep ${OCF_RESKEY_op_sleep}
        rm ${VERIFY_SERIALIZED_FILE}
    fi

    if [ -f ${OCF_RESKEY_state} ]; then
        return $OCF_SUCCESS
    fi
    if false ; then
        return $OCF_ERR_GENERIC
    fi
    return $OCF_NOT_RUNNING
}

eio_validate() {

    # Is the state directory writable? 
    state_dir=`dirname "$OCF_RESKEY_state"`
    touch "$state_dir/$$"
    if [ $? != 0 ]; then
    return $OCF_ERR_ARGS
    fi
    rm "$state_dir/$$"

    return $OCF_SUCCESS
}

: ${OCF_RESKEY_fake=eio}
: ${OCF_RESKEY_op_sleep=0}
: ${OCF_RESKEY_CRM_meta_interval=0}
: ${OCF_RESKEY_CRM_meta_globally_unique:="true"}

if [ "x$OCF_RESKEY_state" = "x" ]; then
    if [ ${OCF_RESKEY_CRM_meta_globally_unique} = "false" ]; then
    state="${HA_VARRUN}/EIO-${OCF_RESOURCE_INSTANCE}.state"

    # Strip off the trailing clone marker
    OCF_RESKEY_state=`echo $state | sed s/:[0-9][0-9]*\.state/.state/`
    else 
    OCF_RESKEY_state="${HA_VARRUN}/EIO-${OCF_RESOURCE_INSTANCE}.state"
    fi
fi
VERIFY_SERIALIZED_FILE="${OCF_RESKEY_state}.serialized"

case $__OCF_ACTION in
meta-data)  meta_data
        exit $OCF_SUCCESS
        ;;
start)      eio_start;;
stop)       eio_stop;;
monitor)    eio_monitor;;
migrate_to) ocf_log info "Migrating ${OCF_RESOURCE_INSTANCE} to ${OCF_RESKEY_CRM_meta_migrate_target}."
            eio_stop
        ;;
migrate_from)   ocf_log info "Migrating ${OCF_RESOURCE_INSTANCE} to ${OCF_RESKEY_CRM_meta_migrate_source}."
            eio_start
        ;;
reload)     ocf_log err "Reloading..."
            eio_start
        ;;
validate-all)   eio_validate;;
usage|help) eio_usage
        exit $OCF_SUCCESS
        ;;
*)      eio_usage
        exit $OCF_ERR_UNIMPLEMENTED
        ;;
esac
rc=$?
ocf_log debug "${OCF_RESOURCE_INSTANCE} $__OCF_ACTION : $rc"
exit $rc
#!/bin/bash
#
# chkconfig: 35 90 12
# description: Foo server
#
# Get function from functions library
. /etc/init.d/functions
# Start the service FOO
start() {
        initlog -c "echo -n Starting EIO for samba_cache: "
    #/bin/cp -rf /opt/94/* /etc/udev/rules.d/
    sleep 2s
        eio_cli enable -d /dev/vgdrbd11/lvdrbd_samba -s /dev/vgdrbd10/lvdrbd_cache -p lru -m wb -c samba_cache
        #eio_cli create -d /dev/vgdrbd11/lvdrbd_samba -s /dev/vgdrbd10/lvdrbd_cache -p lru -m wb -c samba_cache
        ### Create the lock file ###
        touch /var/lock/subsys/eio_samba_cache
        success $"Cache samba_cache started"
        echo
}
# Restart the service FOO
stop() {
        initlog -c "echo -n Stopping EIO for samba_cache: "
    eio_cli edit -c samba_cache -m ro
    grep "nr_dirty                              0" /proc/enhanceio/samba_cache/stats
    while [ $? -ne 0]; do
        grep "nr_dirty                              0" /proc/enhanceio/samba_cache/stats
    done
    eio_cli delete -c samba_cache
        ### Now, delete the lock file ###
        rm -f /var/lock/subsys/eio_samba_cache
        echo
}
### main logic ###
case "$1" in
  start)
        start
        ;;
  stop)
        stop
        ;;
  status)
        status FOO
        ;;
  restart|reload|condrestart)
        stop
        start
        ;;
  *)
        echo $"Usage: $0 {start|stop|restart|reload|status}"
        exit 1
esac
exit 0
```
