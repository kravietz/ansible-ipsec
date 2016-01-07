# Securing cloud servers with IPSec and Ansible

This set of [Ansible](http://docs.ansible.com/index.html) templates is an example how IPSec suite of protocols can be used to simplify transport security in networks of Linux-based cloud servers. It's not ready to use, drop-in Ansible plugin, more of a template,  but should be relatively easy to adapt to a network of any complexity.

The repository contains two git branches: `master` which only produces **local** configuration files with lots of debugging output (including plaintext secrets) without atually touching any of the remote servers listed in the example `inventory`. It's safe to run even in the local repo. The `prod` branch will actually install configuration files on remote servers, restart services etc and it won't work unmodified on your network.

## Quick start (Racoon)
* Make sure you have Ansible installed (`pip install ansible` or your Linux distro's equivalent)
* Checkout the repo:

    git checkout https://github.com/kravietz/ansible-ipsec.git

* Run the `racoon.yml` playbook to generate Racoon configuration files for IKE-managed IPSec coniguration:

    cd ansible-ipsec
    ansible-playbook -i inventory racoon.yml

The files will generated into the `output/racoon` directory, just as if they would be placed in `/etc`. The following files will be produced:

    output/racoon/web1/racoon/racoon.conf
    output/racoon/web1/racoon/pks.txt
    output/racoon/web1/ipsec-tools.conf
    output/racoon/web2... etc


## Quick start (manual keying)
Run the `manual.yml` playbook to generate manual-keyed IPSec configuration:

    ansible-playbook -i inventory manual.yml

The files will be found in `output/manual` directory:

    output/manual/web1/ipsec-tools.conf
    output/manual/web1/ipsec-tools.d/web2
    output/manual/web1/ipsec-tools.d/web3... etc
    output/manual/web2... etc

For manually-keyed configuration the templates will be saved in `ipsec-tools.d` subdirectory, one file for each remote server, to keep them small and more readable. The `prod` mode here works just as in the Racoon mode, except that Racoon configuration files and `racoon` service are not changed.

## Production mode

If you now switch to the `prod` branch (`git checkout prod`) and run the same playfiles as above, it will actually deploy the generated files to `/etc` on servers listed in the `inventory` as  well as restart `racoon` and `setkey` services (this will be the moment of truth). It is also expected that the hostnames in `inventory` are working SSH aliases in production mode. The templates will use default IPv4 addresses of each of the listed hosts.

