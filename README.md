# Host-to-host transport-mode IPSec

Ansible role to enable IPSec encryption between Ansible-managed nodes with minimal performance
overhead. This role is especially suitable for protecting communications between farms of
cloud servers and can effectively replace the need for the complexity of configuring TLS for
each service running on the servers.

## Confiuration

Master IPSec secret, used as seed to securely generate unique pre-shared key for each host pair.
This remains the same across all Ansible managed hosts. Use `ansible-vault` for secure storage.

    ipsec_secret: '088d7633c620f24... generate your own with openssl rand -hex 60'

Should IPSec work in fail-close or fail-open mode? 
* 'use' - fail open: if IPSec cannot be established, the traffic will flow unencrypted.
* 'require' - fail-close: not traffic will be allowed without IPSec.

    ipsec_policy: 'use'

Keying method.
* 'ike' is the preferred keying mode with IKE daemon managing keys and refreshing them at proper
   intervals, suitable for long-term production environments
* 'setkey' uses day-dependent static keys which is **insecure** in long term but may be suitable for
  development environments with frequent Ansible builds that will replace the keys

    ipsec_mode: 'ike'

Never require IPSec for SSH. The assumption is that SSH provides trusted channel on its own and 
it allows remote access to IPSec-enabled servers even if something goes wrong with IPSec channels.
        
    ipsec_open_ssh: yes

Never require IPSec for ICMP protocol. 

    ipsec_open_icmp: yes


