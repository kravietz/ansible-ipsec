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

### Skip SSH
Never require IPSec for SSH. The assumption is that SSH provides trusted channel on its own and 
it allows remote access to IPSec-enabled servers even if something goes wrong with IPSec channels.
        
    ipsec_open_ssh: yes

### Skip ICMP
Never require IPSec for ICMP protocol. This allows network troubleshooting messages such as ping
or port unreachable still work between IPSec-enabled hosts.

    ipsec_open_icmp: yes

### Forwarding
Create IPSec forwarding policies in addition to input and output policies. This is normally only
needed if you use Docker and other such solutions sending traffic through virtual network interfaces
that IPSec will consider forwarded traffic. Not needed for regulard host-to-host traffic and 
disabled by default.

    ipsec_forward: no
    
Note that forwarding traffic only works with IKE keying method and it *may* be tricky as it's more
things that can go wrong between the virtual interfaces (e.g. routing).

### Compression
By default IPCOMP (IPSec compression) is enabled as it will bring performance improvement for textual
data such as SQL and other typical web backend data transfers. When transferring data that is already
compressed, encrypted or otherwise high-entropy, disable compression as it will increase CPU usage
and hurt performance.

    ipsec_compress: yes
    
Note compression only works in IKE mode.

## How are keys derived?

### ipsec_secret
The `ipsec_secret` constant is a master secret from which all pre-shared secrets for `ike` mode and keys for 'setkey'
more are generated. The master secret only lives on the deployment server running Ansible and should be protected
using [Ansible Vault](https://docs.ansible.com/ansible/latest/cli/ansible-vault.html) or similar secret management
solutions.

Keys for both `ike` and `setkey` mode are derived from the `ipsec_secret` mixed with endpoint hostnames hashed
using SHA256 (the strongest hashing function available in `Jinja2` templates). This technique of deterministic
key generation is used to make sure we can create keys that are identical between each pair of servers in the Ansible
playbook run, but at the same time each pair has a key that is unique and different from others. 

### IKE mode
For **ike** mode keys are stored in the `/etc/racoon/psk.txt` file and they are long term keys generated
using the following `Jinja2` syntax:

```
(hostname, inventory_hostname, ipsec_secret) | sort | join | hash('sha256')
```
Note that in `ike` mode the pre-shared key (PSK) is only used for authentication of the two parties and is never
directly used to encrypt the traffic. After authenticating both ends using PSK, IKE daemon (`racoon`) will then
securely exchange session keys for ESP and periodically renegotiate them. The IKE protocol can effectively
manage ESP keys for thousands of SAs at the same time and takes a number of cryptographic precautions to protect
the PSK in the process, so to the best of my knowledge this key generation method is safe for long-term usage
in production environments.

### Setkey mode 

The **setkey** mode uses manual keying in which we need to configure actual raw encryption keys used for
encryption and authentication of ESP packats. Because we're keying individual SAD entries, for each host pair
we must produce unique keys for ESP (encryption) and MAC (authentication) separately. and then repeat that
for return traffic. In addition, non-secret connection identifiers (SPI and IPC) need to be generated in the same
manner. 

Key purpose diversity is acquired by mixing in static strings ("ESP", "MAC" etc). Then we also add current day
number so keys will be cycled on subsequent Ansible runs. Obviously, this is much less secure than IKE.
```
run_date=(template_run_date['year'], template_run_date['month'], template_run_date['day'])
( run_date, host1 , host2 , ipsec_secret , "ESP" )   | sort | join | hash('sha256') | truncate(32,end='')
( run_date, host1 , host2 , ipsec_secret , "MAC" )   | sort | join | hash('sha256') | truncate(64,end='')
( run_date, host1 , host2 , ipsec_secret , "SPI" )   | sort | join | hash('sha256') | truncate(6,end='')
( run_date, host1 , host2 , ipsec_secret , "IPC" )   | sort | join | hash('sha256') | truncate(6,end='')
```
