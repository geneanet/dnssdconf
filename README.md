# dnssdconf
Generate configuration files based on DNS-SD records

When you manage clusters of servers, you often have to update configuration files when adding or removing machines, be it on the loadbalancer that forward your trafic to the backends, or on the deployment system that push your code on the servers.
Having to change manually several configuration files implies that you can make errors or even forget to update one.
To simplify this task, dnssdconf allows to use the DNS Service Discovery protocol to fetch the list of backend servers and generate configuration files based on templates.
Thus, adding or removing a server in a cluster is as simple as changing one DNS record.

Adding dnssdconf in a crontab run every minute will ensure that your configuration files are always up to date.

### Table of contents

<!-- TOC depthFrom:2 depthTo:2 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Usage](#usage)
- [Config file](#config-file)
- [Templates](#templates)
- [DNS-SD records](#dns-sd-records)
- [Dependencies](#dependencies)

<!-- /TOC -->

## Usage
__dnssdconf__ [--logfile LOGFILE] [--loglevel {debug,info,warning,error,critical,fatal}] [--logconfig LOGCONFIG] [--config CONFIG] [--templatesdir TEMPLATESDIR] [--diff]

Arguments :
* __--logfile__ : Log file
* __--loglevel__ : Log level, choose between the following values :debug,info,warning,error,critical,fatal
* __--logconfig__ : Logging configuration file (overrides --loglevel and --logfile). File format is described at https://docs.python.org/2/library/logging.config.html#logging-config-fileformat.
* __--config__ : Config file, describing which files to generate and what DNS-SD services to fetch (default: /etc/dnssdconf/dnssdconf).
* __--templatesdir__ : Default templates directory (default: /etc/dnssdconf/template)
* __--diff__ : Show diff when files are updated (loglevel info)
* __--backup [suffix]__ : Backup files with this suffix
* __--noop__ : Do not write files, only pretend to
* __--allowemptyconf__ : Do not emit a warning if dnssdconf config file is emty

## Config file
The config file is a YAML file of which each main element is the description of a file to generate.
Each file can have several properties :
* __service_instances__ : map an identifier to a DNS name for each service instance to be fetched.
* __template__ : path to the template. If it does not begin with a /, it is considered to be relative to the path specified with the _--templatesdir_ CLI option.
* __output__ : path to the output file. It it does not begin with a /, it is considered to be relative to the current working directory.
* __exec__ : command to be executed after the file is updated (only if the content has changed). The command is passed to a shell.
* __encoding__ : set the encoding of the template and output file. Default is _ascii_. Supported values are described at https://docs.python.org/2/library/codecs.html#standard-encodings.
* __resolve_names__ : resolve names pointed by SRV records. If the value is True or 'any', resolve A and AAAA records, fail if both are not defined; if the value is 'both', resolve A and AAAA records, fail if any is not defined; if the value is 'ipv4' or 'ipv6', resolve respectively A and AAAA record and fail if it is not defined. Defaults to False.

### Example:

    haproxy:
        output: /etc/haproxy/haproxy.cfg
        template: haproxy.conf.mako
        exec: service haproxy reload
        service_instances:
            web: web._http._tcp.mycompany.example
            bdd: main._mysql._tcp.mycompany.example
    nginx:
        output: /etc/nginx/nginx.conf
        [...]

## Templates
The templates are rendered using the Mako template engine (http://docs.makotemplates.org/).
Services instances are available as variables using the identifier set in the configuration file.

A service instance has two properties :
* __servers__ : a dictionary representing the servers, each having the following properties :
    * hostname
    * port
    * priority
    * weight
    * ipv4 (defined if resolve_names is True, 'any', 'both' or 'ipv4' and an A record exists for the server)
    * ipv6 (defined if resolve_names is True, 'any', 'both' or 'ipv6' and an AAAA record exists for the server)
    * __config__ : a dictionary representing the instance properties, as defined in the TXT record associated with the service instance.

Example (for a haproxy configuration file, as the one in the example above) :

    listen web
            bind :80
    % for server in web.servers:
            server ${server.hostname.split('.')[0]} ${server.hostname}:${server.port} check inter 2000 fall 5 rise 2 maxconn ${server.weight * 2} weight ${server.weight}
    % endfor

In this example, we used python expressions to extract a short host name from the FQDN, and to compute a connection limit from the weight.

## DNS-SD records
The DNS-SD records must respect the format defined in the RFC 6763 (https://tools.ietf.org/html/rfc6763).

Each service instance must have at least :
* One TXT record, which may be empty.
* At least one SRV record

Example :

    web._http._tcp.mycompany.example.      IN      TXT     ""
    web._http._tcp.mycompany.example.      IN      SRV     0       12      80      server1.mycompany.example.
    web._http._tcp.mycompany.example.      IN      SRV     0       27      80      server2.mycompany.example.
    web._http._tcp.mycompany.example.      IN      SRV     0       17      80      server3.mycompany.example.

## Dependencies

dnssdconf depends on the following python libraries :
* __dnspython__ (http://www.dnspython.org/)
* __mako__ (http://www.makotemplates.org/)
* __pyyaml__ (http://pyyaml.org/)

Under debian/ubuntu, they can be installed using the following packages : python-dnspython, python-mako and python-yaml.
