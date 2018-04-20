#!/bin/sh
#
#
#   DM-WRITEBOOST OCF RA. Does nothing but wait a few seconds, can be
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
<resource-agent name="writeboost" version="1.0">
<version>1.0</version>

<longdesc lang="en">
dm-writeboost cache
</longdesc>
<shortdesc lang="en">writeboost</shortdesc>

<parameters>
<parameter name="state" unique="1">
<longdesc lang="en">
Location to store the resource state in.
</longdesc>
<shortdesc lang="en">State file</shortdesc>
<content type="string" default="${HA_VARRUN}/WRITEBOOST-{OCF_RESOURCE_INSTANCE}.state" />
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
<content type="string" default="writeboost" />
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
<action name="start"        timeout="3600" />
<action name="stop"         timeout="1111" />
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

writeboost_usage() {
    cat <<END
usage: $0 {start|stop|monitor|migrate_to|migrate_from|validate-all|meta-data}

Expects to have a fully populated OCF RA-compliant environment set.
END
}

writeboost_start() {
    /sbin/writeboost
    touch ${OCF_RESKEY_state}
    return $OCF_SUCCESS
}

writeboost_stop() {
    /sbin/writeboost -u
    rm ${OCF_RESKEY_state}
    return $OCF_SUCCESS
}

writeboost_monitor() {
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

writeboost_validate() {

    # Is the state directory writable? 
    state_dir=`dirname "$OCF_RESKEY_state"`
    touch "$state_dir/$$"
    if [ $? != 0 ]; then
    return $OCF_ERR_ARGS
    fi
    rm "$state_dir/$$"

    return $OCF_SUCCESS
}

: ${OCF_RESKEY_fake=writeboost}
: ${OCF_RESKEY_op_sleep=0}
: ${OCF_RESKEY_CRM_meta_interval=0}
: ${OCF_RESKEY_CRM_meta_globally_unique:="true"}

if [ "x$OCF_RESKEY_state" = "x" ]; then
    if [ ${OCF_RESKEY_CRM_meta_globally_unique} = "false" ]; then
    state="${HA_VARRUN}/WRITEBOOST-${OCF_RESOURCE_INSTANCE}.state"

    # Strip off the trailing clone marker
    OCF_RESKEY_state=`echo $state | sed s/:[0-9][0-9]*\.state/.state/`
    else 
    OCF_RESKEY_state="${HA_VARRUN}/WRITEBOOST-${OCF_RESOURCE_INSTANCE}.state"
    fi
fi
VERIFY_SERIALIZED_FILE="${OCF_RESKEY_state}.serialized"

case $__OCF_ACTION in
meta-data)  meta_data
        exit $OCF_SUCCESS
        ;;
start)      writeboost_start;;
stop)       writeboost_stop;;
monitor)    writeboost_monitor;;
migrate_to) ocf_log info "Migrating ${OCF_RESOURCE_INSTANCE} to ${OCF_RESKEY_CRM_meta_migrate_target}."
            writeboost_stop
        ;;
migrate_from)   ocf_log info "Migrating ${OCF_RESOURCE_INSTANCE} to ${OCF_RESKEY_CRM_meta_migrate_source}."
            writeboost_start
        ;;
reload)     ocf_log err "Reloading..."
            writeboost_start
        ;;
validate-all)   writeboost_validate;;
usage|help) writeboost_usage
        exit $OCF_SUCCESS
        ;;
*)      writeboost_usage
        exit $OCF_ERR_UNIMPLEMENTED
        ;;
esac
rc=$?
ocf_log debug "${OCF_RESOURCE_INSTANCE} $__OCF_ACTION : $rc"
exit $rc
