# <a id="service-monitoring"></a> Service Monitoring

The power of Icinga 2 lies in its modularity. There are thousands of
community plugins available next to the standard plugins provided by
the [Monitoring Plugins project](https://www.monitoring-plugins.org).

## <a id="service-monitoring-requirements"></a> Requirements

### <a id="service-monitoring-plugins"></a> Plugins

All existing Nagios or Icinga 1.x plugins work with Icinga 2. Here's a
list of popular community sites which host check plugins:

* [Icinga Exchange](https://exchange.icinga.org)
* [Icinga Wiki](https://wiki.icinga.org)

The recommended way of setting up these plugins is to copy them to a common directory
and create a new global constant, e.g. `CustomPluginDir` in your [constants.conf](4-configuring-icinga-2.md#constants-conf)
configuration file:

    # cp check_snmp_int.pl /opt/monitoring/plugins
    # chmod +x /opt/plugins/check_snmp_int.pl

    # cat /etc/icinga2/constants.conf
    /**
     * This file defines global constants which can be used in
     * the other configuration files. At a minimum the
     * PluginDir constant should be defined.
     */

    const PluginDir = "/usr/lib/nagios/plugins"
    const CustomPluginDir = "/opt/monitoring/plugins"

Prior to using the check plugin with Icinga 2 you should ensure that it is working properly
by trying to run it on the console using whichever user Icinga 2 is running as:

    # su - icinga -s /bin/bash
    $ /opt/monitoring/plugins/check_snmp_int.pl --help

Additional libraries may be required for some plugins. Please consult the plugin
documentation and/or plugin provided README for installation instructions.
Sometimes plugins contain hard-coded paths to other components. Instead of changing
the plugin it might be easier to create logical links which is (more) update-safe.

Sometimes there are plugins which do not exactly fit your requirements.
Either you'll modify an existing plugin or you'll write one by yourself.

### <a id="service-monitoring-plugin-checkcommand"></a> CheckCommand Definition

Each plugin requires a [CheckCommand](9-object-types.md#objecttype-checkcommand) object in your
configuration which can be used in the [Service](9-object-types.md#objecttype-service) or
[Host](9-object-types.md#objecttype-host) object definition.

Please check if the Icinga 2 package already provides an
[existing CheckCommand definition](10-icinga-template-library.md#plugin-check-commands).
If yes throroughly check the required parameters and integrate the check command
into your host and service objects.

There are the following conventions to follow when adding a new command object definition:

* Always import the `plugin-check-command` template
* Use [command-arguments](#) whenever possible. The `command` attribute must be an array
in `[ ... ]` then for shell escaping.
* Define a unique `prefix` for the command's specific command arguments. That way you can safely
set them on host/service level and you'll always know which command they control.
* Use command argument default values, e.g. for thresholds
* Use [advanced conditions](9-object-types.md#objecttype-checkcommand) like `set_if` definitions.

Example for a custom `my-snmp-int` check command:

    object CheckCommand "my-snmp-int" {
      import "plugin-check-command"

      command = [ CustomPluginDir + "/check_snmp_int.pl" ]

      arguments = {
        "-H" = "$snmp_address$"
        "-C" = "$snmp_community$"
        "-p" = "$snmp_port$"
        "-2" = {
          set_if = "$snmp_v2$"
        }
        "-n" = "$snmp_interface$"
        "-f" = {
          set_if = "$snmp_perf$"
        }
        "-w" = "$snmp_warn$"
        "-c" = "$snmp_crit$"
      }

      vars.snmp_v2 = true
      vars.snmp_perf = true
      vars.snmp_warn = "300,400"
      vars.snmp_crit = "0,600"
    }


For further information on your monitoring configuration read the
[Monitoring Basics](3-monitoring-basics.md#monitoring-basics) chapter.

If you have created your own `CheckCommand` definition please kindly
[send it upstream](https://wiki.icinga.org/display/community/Contribute+Icinga+2+ITL+Plugin+Check+Command+Definitions)

### <a id="service-monitoring-plugin-api"></a> Plugin API

Currently Icinga 2 supports the native plugin API specification from the `Monitoring Plugins`
project.

The `Monitoring Plugin API` is defined in the [Monitoring Plugins Development Guidelines](https://www.monitoring-plugins.org/doc/guidelines.html).

## <a id="service-monitoring-overview"></a> Service Monitoring Overview

The following examples should get you started with your own integration ideas.
There is a variety of common plugins available. This collection is not complete --
if you have any updates please send a documentation patch upstream.

## <a id="service-monitoring-general"></a> General Monitoring

If the remote service is available using a network protocol and port,
and a check plugin is available, you don't necessarily need a local client installed.
Rather choose a plugin and configure all parameters and thresholds. The [Icinga 2 Template Library](10-icinga-template-library.md#icinga-template-library)
already ships various examples like

* [ping4](10-icinga-template-library.md#plugin-check-command-ping4), [ping6](10-icinga-template-library.md#plugin-check-command-ping6),
[fping4](10-icinga-template-library.md#plugin-check-command-fping4), [fping6](10-icinga-template-library.md#plugin-check-command-fping6), [hostalive](10-icinga-template-library.md#plugin-check-command-hostalive)
* [tcp](10-icinga-template-library.md#plugin-check-command-tcp), [udp](10-icinga-template-library.md#plugin-check-command-udp), [ssl](10-icinga-template-library.md#plugin-check-command-ssl)
* [ntp_time](10-icinga-template-library.md#plugin-check-command-ntp-time)

## <a id="service-monitoring-linux"></a> Linux Monitoring

* [disk](10-icinga-template-library.md#plugin-check-command-disk)
* [mem](10-icinga-template-library.md#plugin-contrib-command-mem), [swap](10-icinga-template-library.md#plugin-check-command-swap)
* [running_kernel](10-icinga-template-library.md#plugin-contrib-command-running_kernel)
* Package repositores ([apt](10-icinga-template-library.md#plugin-check-command-apt), [yum](10-icinga-template-library.md#plugin-contrib-command-yum), etc.)
* [ssh](10-icinga-template-library.md#plugin-check-command-ssh)
* Performance ([iostat](10-icinga-template-library.md#plugin-contrib-command-iostat), [check_sar_perf](https://github.com/dnsmichi/icinga-plugins/blob/master/scripts/check_sar_perf.py))

## <a id="service-monitoring-windows"></a> Windows Monitoring

* [check_wmi_plus](http://www.edcint.co.nz/checkwmiplus/)
* [NSClient++](https://www.nsclient.org) (in combination with the Icinga 2 client as [nscp-local](10-icinga-template-library.md#nscp-plugin-check-commands) check commands)
* [Icinga 2 Windows Plugins](10-icinga-template-library.md#windows-plugins) (disk, load, memory, network, performance counters, ping, procs, service, swap, updates, uptime, users
* vbs and Powershell scripts

## <a id="service-monitoring-database"></a> Database Monitoring

* MySQL/MariaDB: [mysql_health](10-icinga-template-library.md#plugin-contrib-command-mysql_health), [mysql](10-icinga-template-library.md#plugin-check-command-mysql), [mysql_query](10-icinga-template-library.md#plugin-check-command-mysql-query)
* PostgreSQL: [postgres](10-icinga-template-library.md#plugin-contrib-command-postgres)
* Oracle: [oracle_health](10-icinga-template-library.md#plugin-contrib-command-oracle_health)
* MSSQL: [mssql_health](10-icinga-template-library.md#plugin-contrib-command-mssql_health)
* DB2: [db2_health](10-icinga-template-library.md#plugin-contrib-command-db2_health)
* MongoDB: [mongodb](10-icinga-template-library.md#plugin-contrib-command-mongodb)
* Elasticsearch: [elasticsearch](10-icinga-template-library.md#plugin-contrib-command-elasticsearch)
* Redis: [redis](10-icinga-template-library.md#plugin-contrib-command-redis)

## <a id="service-monitoring-snmp"></a> SNMP Monitoring

* [Manubulon plugins](10-icinga-template-library.md#snmp-manubulon-plugin-check-commands) (interface, storage, load, memory, process)
* [snmp](10-icinga-template-library.md#plugin-check-command-snmp), [snmpv3](10-icinga-template-library.md#plugin-check-command-snmpv3)

## <a id="service-monitoring-network"></a> Network Monitoring

* [nwc_health](10-icinga-template-library.md#plugin-contrib-command-nwc_health)
* [interfaces](10-icinga-template-library.md#plugin-contrib-command-interfaces)
* [interfacetable](10-icinga-template-library.md#plugin-contrib-command-interfacetable)
* [iftraffic](10-icinga-template-library.md#plugin-contrib-command-iftraffic), [iftraffic64](10-icinga-template-library.md#plugin-contrib-command-iftraffic64)

## <a id="service-monitoring-web"></a> Web Monitoring

* [http](10-icinga-template-library.md#plugin-check-command-http)
* [ftp](10-icinga-template-library.md#plugin-check-command-ftp)
* [webinject](10-icinga-template-library.md#plugin-contrib-command-webinject)
* [squid](10-icinga-template-library.md#plugin-contrib-command-squid)
* [apache_status](10-icinga-template-library.md#plugin-contrib-command-apache_status)
* [nginx_status](10-icinga-template-library.md#plugin-contrib-command-nginx_status)
* [kdc](10-icinga-template-library.md#plugin-contrib-command-kdc)
* [rbl](10-icinga-template-library.md#plugin-contrib-command-rbl)

## <a id="service-monitoring-java"></a> Java Monitoring

* [jmx4perl](10-icinga-template-library.md#plugin-contrib-command-jmx4perl)

## <a id="service-monitoring-dns"></a> DNS Monitoring

* [dns](10-icinga-template-library.md#plugin-check-command-dns)
* [dig](10-icinga-template-library.md#plugin-check-command-dig)
* [dhcp](10-icinga-template-library.md#plugin-check-command-dhcp)

## <a id="service-monitoring-backup"></a> Backup Monitoring

* [check_bareos](https://github.com/widhalmt/check_bareos)

## <a id="service-monitoring-log"></a> Log Monitoring

* [check_logfiles](https://labs.consol.de/nagios/check_logfiles/)
* [check_logstash](https://github.com/widhalmt/check_logstash)
* [check_graylog2_stream](https://github.com/Graylog2/check-graylog2-stream)

## <a id="service-monitoring-virtualization"></a> Virtualization Monitoring

## <a id="service-monitoring-virtualization-vmware"></a> VMWare Monitoring

* [esxi_hardware](10-icinga-template-library.md#plugin-contrib-command-esxi-hardware)
* [VMWare](10-icinga-template-library.md#plugin-contrib-vmware)

**Tip**: If you are encountering timeouts using the VMWare Perl SDK
check [this blog entry](http://www.claudiokuenzler.com/blog/650/slow-vmware-perl-sdk-soap-request-error-libwww-version).

## <a id="service-monitoring-sap"></a> SAP Monitoring

* [check_sap_health](https://labs.consol.de/nagios/check_sap_health/index.html)
* [SAP CCMS](https://sourceforge.net/projects/nagios-sap-ccms/)

## <a id="service-monitoring-mail"></a> Mail Monitoring

* [smtp](10-icinga-template-library.md#plugin-check-command-smtp), [ssmtp](10-icinga-template-library.md#plugin-check-command-ssmtp)
* [imap](10-icinga-template-library.md#plugin-check-command-imap), [simap](10-icinga-template-library.md#plugin-check-command-simap)
* [pop](10-icinga-template-library.md#plugin-check-command-pop), [spop](10-icinga-template-library.md#plugin-check-command-spop)

## <a id="service-monitoring-hardware"></a> Hardware Monitoring

* [hpasm](10-icinga-template-library.md#plugin-contrib-command-hpasm)
* [ipmi-sensor](10-icinga-template-library.md#plugin-contrib-command-ipmi-sensor)

## <a id="service-monitoring-metrics"></a> Metrics Monitoring

* [graphite](10-icinga-template-library.md#plugin-contrib-command-graphite)
