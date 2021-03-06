# XDMoD Open OnDemand Module

The XDMoD Open OnDemand Module is an optional module for
tracking usage of the Open OnDemand software in XDMoD.

For more information about Open OnDemand please visit
[the Open OnDemand website](https://openondemand.org/)

For more information, including information about additional Open XDMoD
capabilities provided as optional modules, please visit
[the Open XDMoD website](https://open.xdmod.org).

The OnDemand module provides an "OnDemand" realm in XDMoD.
This OnDemand realm displays information about usage of Open OnDemand
from the web server logs.

The default permission is that user accounts with center director and center
staff role have full access to the OnDemand realm. Other accounts and the
public user will not see the realm.

# External Dependencies

The OnDemand module can use a GeoLite2 database to display location
information based on the IP address from the webserver logs. The IP to
location mapping is performed at data ingest time. The code has been
tested with [MaxMind's GeoLite2 City database](https://dev.maxmind.com/geoip/geoip2/geolite2/).

The database file is not distributed with Open XDMoD and must be
obtained separately. If no database is present then all location
information will be marked as 'Unknown'. The database is not
required for the Open XDMoD module to display and process OnDemand
server log data.

# Install and configuration

The OnDemand module should be added to an existing working instance of
Open XDMoD version 9.5 or later. See the [Open XDMoD site](https://open.xdmod.org/9.5/)
for setup and install instructions.

## Source code install

The source package is installed as follows:

    $ tar zxvf xdmod-ondemand-9.5.0.tar.gz
    $ cd xdmod-ondemand-9.5.0
    # ./install --prefix=/opt/xdmod

Change the prefix as desired. The default installation prefix is /usr/local/xdmod. These instructions assume you are installing Open XDMoD in /opt/xdmod.

## RPM install

    # yum install xdmod-ondemand-9.5.0-1.0.el7.noarch.rpm

## Configuration

### Interactive script configuration

The Open OnDemand XDMoD module adds an additional main menu item to the XDMoD interactive setup software. Run the script as follows:

    # xdmod-setup

and select the ‘Open OnDemand’ option in the main menu. The Open OnDemand
section only has a single option: setup the database.  This option creates the
necessary database schema in the XDMoD
datawarehouse. You will need to provide the credentials for your MySQL root
user, or another user that has privileges to create databases. A single
database `modw_ondemand` will be created.  The database user that is
specified in the `portal_settings.ini` will be granted access to this
database.

### Manual configuration

If the database server is located on a different host than the webserver then it is necessary
to setup the database manually.

Create a database schema called `modw_ondemand` and grant permission for the XDMoD database user
account to access this schema.

Once the schema is created then the `acl-config` command should be run:

    $ /usr/xdmod/bin/acl-config

### Resource Setup

Add a new resource to Open XDMoD using the `xdmod-setup` script.
Instructions for adding the resource are on the [main Open XDMoD page](https://open.xdmod.org/9.5/configuration.html#resources)

The Open OnDemand resource must have a type set to `gateway`.

The resource setup menu will prompt for the node and core count for the resource. These
data are not currently used by the On Demand module so any value can be used.

After the resource has been added then the `xdmod-ingestor` script must be run to load
the resource information into the XDMoD datawarehouse.


### GeoIP database setup

The module infers geographic location based on the IP address in the Open OnDemand
logs. If a GeoLite2 City database file is available then it should be copied
to a path where the `xdmod` user has read access.

The `/etc/xdmod/portal_settings.d/ondemand.ini` configuration
file should be edited to add the path to the file to the `geoip_database` option. The full path
to the file should be used.

If a GeoLite2 City database file is not available then the `geoip_database` option
should be set to an empty string.

# Usage

Prerequisites:
1) The database schema has been created (either manually or via `xdmod-setup`).
2) The `acl-config` command has been run (`xdmod-setup` does this automatically after the database is setup).
3) The OnDemand resource has been added via `xdmod-setup` and `xdmod-ingestor` run to load the resource
   into XDMoD's datawarehouse.
4) The `portal_settings.d/ondemand.ini` configuration file updated to specify the path to a GeoLite2 City database file.


The OnDemand weblog ingestion pipeline requires three parameters:

| Parameter Name | Description
| -------------- | -----------
| `-r` or `--resource` | Must be set to the name of the resource when it was added to XDMoD in the `xdmod-setup` command. |
| `-u` or `--url` | Must be set to the hostname of the ondemand instance exactly as it appears in the server logs. This includes the `https://` parts and any port numbers but do not include the trailing forward slash. |
| `-d` or `--dir` | Set to the path to a directory containing webserver log files from the Open OnDemand server. The ingestor will process all files in this directory that have the suffix `.log` or `.log.X` where X is a number |


The pipeline should be run as the `xdmod` user as follows:

    xdmod-ondemand-ingestor -d /path/to/ood_server_logs -r [resource] -u [ondemand hostname]

### Hints

For log files with a large amount of data (hundreds of thousands of lines), the ingestion pipeline
will run faster if you split large log files into smaller ones. An example of how to do this
is to use the `split` commandline tool to split the large logf file by lines and generate
output files with a numbered suffix (note the period at the end of the output filename):

```bash
split -d -l 30000 [LARGE INPUT FILE] /scratch/ondemand/webserver.log.
```

### FAQ

#### The OnDemand realm does no show in the XDMoD portal

Make sure you are logged in with a user account that has either Center Director
or Center Staff role. Note the admin user account that is created with `xdmod-setup`
 does not have these roles by default.

#### The xdmod-ondemand-ingestor command ran without error, but no data

Re-run the `xdmod-ondemand-ingestor` command with the `--debug` flag to show detailed information
about what it is running. Things to double check:
- The `webserver_format_str` setting matches the data format in the log files
- The filenames of the log files end in `.log` or `.log.N` where `N` is a number
- The url specified on the command line exactly matches the url in the webserver log files. This should include the protocol part (https://) and any port numbers but not the trailing forward slash character.
