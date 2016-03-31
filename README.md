# dnssdconf
Generate config files based on DNS-SD records

When you manage clusters of servers, you often have to update config files when adding or removing machines, be it on the loadbalancer that forward your trafic to the backends, or on the deployment system that push your code on the servers.
Having to change manually several config files implies that you can make errors or even forget to update one.
To simplify this task, dnssdconf allows to use the DNS Service Discovery protocol to fetch the list of backend servers and generate config files based on templates.
Thus, adding or removing a server in a cluster is as simple as changing one DNS record.

Adding dnssdconf in a crontab run every minute will ensure that your config files are always up to date.
