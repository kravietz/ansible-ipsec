# Host-to-host transport-mode IPSec

Ansible role to enable IPSec encryption between Ansible-managed nodes with minimal performance
overhead. This role is especially suitable for protecting communications between farms of
cloud servers and can effectively replace the need for the complexity of configuring TLS for
each service running on the servers.

It was designed to provide production-level security with ease of management and deployment in
continuous integration environments. The confidentiality of communications depends on a single secret,
stored in Ansible configuration, that is further used to derive protocol-level secrets. Full rekeying
of the whole network can be  achieved as a matter of changing the secret and re-running Ansible playbook.

## Changelog
* v1.1 - full support for IPv6, bugfixes
* v1.0 - initial release

## Inventory

Create group `ipsec` and add all hosts that should be IPSec connected:

    [ipsec]
    host1
    host2
    host3

This role will always create IPSec configuration for *full* `ipsec` group on each host, regardless
of current play scope limitation. This is to ensure that scope-limited runs don't leave some
hosts with IPSec configuration and their counterparts without one, which would cause issues
with the `require` policy.

The role depends on `ansible_default_ipv4` and `ansible_default_ipv6` automatic variables for IP addresses so
[Ansible fact caching](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#fact-caching)
must be enabled.

## Firewall

For IPSec to work the following ports and protocols must be opened:

* `500/udp` IKE (`iptables -A INPUT -p udp --dport 500 -j ACCEPT`)
* `esp` the ESP protocol (`iptables -A INPUT -p esp -j ACCEPT`)

These ports should be only opened to the other IPSec peers, there's no need to open them
publicly.

## Configuration

The role uses the following variables that should be set for the whole `ipsec` group (`group_vars/ipsec`).
See [Ansible variables](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#variable-examples)
for guidance on setting variables.

### Master secret

Master IPSec secret, used as seed to securely generate unique pre-shared key for each host pair.
This remains the same across all Ansible managed hosts. **You must customize this or the whole exercies 
will make little sense!**

    ipsec_secret: '088d7633c620f24... generate your own with openssl rand -hex 60'
    
Always use [Ansible Vault](https://docs.ansible.com/ansible/latest/cli/ansible-vault.html) to keep the group
variables files encrypted.

### Address families
By default SAD/SPD entries will be created for both IPv4 and IPv6. If either of them is not needed, you can delete
it here but make sure it remains a list. These are Ansible variable names containing IPv4 and IPv6 address
of the default interface collected during [fact caching](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#fact-caching). 

    ipset_inet:
    - 'ansible_default_ipv6'
    - 'ansible_default_ipv4'

### Traffic encryption policy 
Should IPSec work in fail-close or fail-open mode? 
* `use` - fail open: if IPSec cannot be established, the traffic will flow unencrypted
* `require` - fail closed: not traffic will be allowed without IPSec
* `disable` - disable IPSec policies at all; can be used as a quick off switch

    ipsec_policy: 'use'
    
Note that `disable` will only remove the kernel-level IPSec policies, stopping any attempts to establish
and require IPSec for the current traffic but IKE configuration will remain in place as no-op.

### Keying method
* `ike` is the preferred keying mode with IKE daemon managing keys and refreshing them at proper
   intervals, suitable for long-term production environments
* `setkey` uses day-dependent static keys which is *insecure in long term* but may be suitable for
  development environments with frequent Ansible builds that will replace the keys; the IKE daemon
  is not used, everything happens on the kernel network stack

    ipsec_mode: 'ike'

### Open SSH
Never require IPSec for SSH. The assumption is that SSH provides trusted channel on its own and 
it allows remote access to IPSec-enabled servers even if something goes wrong with IPSec channels.
        
    ipsec_open_ssh: yes

### Open ICMP
Never require IPSec for ICMP protocol. This allows network troubleshooting messages such as ping
or port unreachable still work between IPSec-enabled hosts.

    ipsec_open_icmp: yes

### Forwarding
Create IPSec forwarding policies in addition to input and output policies. This is normally only
needed if you use Docker and other such solutions sending traffic through virtual network interfaces
that IPSec will consider forwarded traffic. Not needed for regulard host-to-host traffic and 
disabled by default.

    ipsec_forward: no
    
Note that forwarding traffic only works with IKE keying method.

## How are keys derived?

Keys for both `ike` and `setkey` mode are derived from the `ipsec_secret` and endpoint hostnames hashed
using SHA256 (the strongest hashing function available in `Jinja2` templates). Mixing the hostnames with
the secret ensures each host pair uses a distinct pre-shared secret for key management and authentication.

For **ike** mode keys are stored in the `/etc/racoon/psk.txt` file and they are long term keys generated
using the following `Jinja2` syntax:

```
(hostname, inventory_hostname, ipsec_secret) | sort | join | hash('sha256')
```
Note that in `ike` mode the pre-shared keu is never used directly in the key exchange process and the IKE protocol
takes a number of cryptographic precautions to protect it so to the best of my knowledge this key generation
method is safe for long-term usage in production environments. 

The **setkey** mode uses manual keying so keys need to be generated for each direction, and for
ESP (encryption) and MAC (authentication) separately. In addition, non-secret connection identifiers
(SPI and IPC) need to be generated for each direction. This is achieved by additionally mixing the 
`ipsec_secret` with static strings, one for each of the required purposes.

Playbook run day number is mixed  into the key input to cycle keys on daily basis (assuming you run
Ansible daily).

```
run_date=(template_run_date['year'], template_run_date['month'], template_run_date['day'])
( run_date, host1 , host2 , ipsec_secret , "ESP" )   | sort | join | hash('sha256') | truncate(32,end='')
( run_date, host1 , host2 , ipsec_secret , "MAC" )   | sort | join | hash('sha256') | truncate(64,end='')
( run_date, host1 , host2 , ipsec_secret , "SPI" )   | sort | join | hash('sha256') | truncate(6,end='')
( run_date, host1 , host2 , ipsec_secret , "IPC" )   | sort | join | hash('sha256') | truncate(6,end='')
```
