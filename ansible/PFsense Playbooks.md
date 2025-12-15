playbooks hre in this document:
1. read the WAN interface IP from a PFSense and save it to an external file
2. read IPSec tunnel info from a PFSense and save it to an external file
3. create IPSec tunnel config (tunnel1 and tunnel2, with phases1 and 2)

xx-xx-xx-xx

- 1) Show the WAN interface of your PFsense and save it to an external file
```yaml
---
- name: Get pfSense Interface Status via SSH
  hosts: fw-pfsense
  gather_facts: no
  tasks:

    - name: Run ifconfig to get interface details
      command: ifconfig pppoe0
      register: ifconfig_output

    - name: Extract WAN Description
      set_fact:
        wan_desc: "{{ ifconfig_output.stdout | regex_search('description: (.+)', '\\1') | default('No Description Found') }}"

    - name: Extract WAN IP Address
      set_fact:
        wan_ip: "{{ (ifconfig_output.stdout | regex_search('inet (\\d+\\.\\d+\\.\\d+\\.\\d+)', '\\1')) | first }}"

    #- name: Show WAN Interface Details
    #  debug:
    #    msg:
    #      - "IFCONFIG: {{ ifconfig_output.stdout_lines }}"
    #      - "WAN Interface Details: {{ wan_desc }}"
    #      - "WAN IP Address: {{ wan_ip }}"

    - name: Write pfsense-vars.yml in the same folder as the playbook
      ansible.builtin.copy:
        content: "wan_ip: {{ wan_ip }}"
        dest: "{{ playbook_dir }}/pfsense-vars.yml"
        mode: '0644'
      delegate_to: localhost
      run_once: true
```

- Format of pfsense-vars.yml file create after run this playbook:
```shell
wan_ip: 102.108.204.205
```


- 2) Show the IPSec Tunnel parameters created on PFSense (IP address local and remote + pre-shard key). Save information on the same file.
```yaml
---
- name: 1. Fetch the config file from remote pfSense host
  hosts: fw-pfsense
  gather_facts: false

  tasks:
    - name: Fetch the pfSense configuration file via SFTP
      ansible.builtin.fetch:
        src: /cf/conf/config.xml
        dest: /tmp/config_files/
        flat: true
      register: config_file_result


- name: 2. Process and Parse the config file locally using Regex (Robust Single-Line Match)
  hosts: localhost
  gather_facts: false
  connection: local

  vars:
    local_config_path: "/tmp/config_files/config.xml"

  tasks:
    - name: Read the local file content into a variable
      ansible.builtin.slurp:
        src: "{{ local_config_path }}"
      register: config_content_encoded

    #- name: debug local file
    #  debug:
    #    var: config_content_encoded


    - name: Decode the content and store it in a fact
      ansible.builtin.set_fact:
        raw_config_text: "{{ config_content_encoded['content'] | b64decode | replace('\\n', '') }}"

    #- name: debug local file decoded
    #  debug:
    #    var: raw_config_text

    - name: Extract all IPSec Phase1 blocks
      set_fact:
        ipsec_phase1_blocks: "{{ raw_config_text | regex_findall('(?s)<phase1>(.*?)</phase1>') }}"

    - name: Initialize tunnel list
      set_fact:
        ipsec_tunnels: []

    - name: Assemble tunnel facts with description
      set_fact:
        ipsec_tunnels: "{{ ipsec_tunnels + [ {
          'name':   (item | regex_search('<descr>(.*?)</descr>', '\\1')) | default(''),
          'local':  (item | regex_search('<myid_data>(.*?)</myid_data>', '\\1')) | default(''),
          'remote': (item | regex_search('<peerid_data>(.*?)</peerid_data>', '\\1')) | default(''),
          'psk':    (item | regex_search('<pre-shared-key>(.*?)</pre-shared-key>', '\\1')) | default('')
        } ] }}"
      loop: "{{ ipsec_phase1_blocks }}"

    - name: Show IPSec tunnel details
      debug:
        msg: >-
          {% for t in ipsec_tunnels %}
          Tunnel "{{ t.name }}":
            Local={{ t.local }}
            Remote={{ t.remote }}
            PSK={{ t.psk }}
          {% endfor %}

    - name: Append IPSec tunnel details in numbered YAML format
      blockinfile:
        path: pfsense-vars.yml
        marker: ""
        block: |
          {% for t in ipsec_tunnels %}
          tunnel{{ loop.index }}:
            name: {{ (t.name | first if (t.name is sequence and t.name is not string) else t.name) }}
            shared_secret: {{ (t.psk | first if (t.psk is sequence and t.psk is not string) else t.psk) }}
            local_ip: {{ (t.local | first if (t.local is sequence and t.local is not string) else t.local) }}
            remote_ip: {{ (t.remote | first if (t.remote is sequence and t.remote is not string) else t.remote) }}
          {% endfor %}
      delegate_to: localhost
```

- Format of pfsense-vars.yml file create after run this playbook:
```shell
wan_ip: 102.108.204.205

tunnel1:
  name: t1_OCI_new
  shared_secret: KsWl52MLCeJMsKI51234567890
  local_ip: 104.104.97.22
  remote_ip: 10.28.15.71
tunnel2:
  name: t2_OCI_new
  shared_secret: SMX6IEuD7OhUa5071234567890
  local_ip: 104.104.97.22
  remote_ip: 10.23.11.73
```

- 3) Playbook to create IPSec Tunnel on Pfsense. It reads the information needed from an external file ../oracle/ipsec-vars.yml.
I'm using the Ansible Collection pfsensible.core (https://galaxy.ansible.com/ui/repo/published/pfsensible/core/docs/)
```yaml
---
- name: PFSense Update IPSec tunnel CONFIG
  hosts: fw-pfsense
  gather_facts: no
  become: true
  collections:
    - pfsensible.core

  vars_files:
    - ../oracle/ipsec-vars.yml   # contains tunnel details / IP and secret

  tasks:
    - name: Read WAN IP address (pppoe0)
      ansible.builtin.shell: "ifconfig pppoe0 | grep 'inet ' | awk '{print $2}'"
      register: wan_ip
      #become: true

    - name: Show WAN IP
      ansible.builtin.debug:
        msg: "WAN (pppoe0) IP is {{ wan_ip.stdout }}"

    - name: Ensure IPSec Phase1 tunnels are updated
      pfsense_ipsec_aggregate:
        # delete all ipsecs before create the new one.
        purge_ipsecs: true
        purge_ipsec_proposals: true
        purge_ipsec_p2s: true

        aggregated_ipsecs:
          - descr: "t1_OCI_new"
            interface: wan
            iketype: ikev2
            remote_gateway: "{{ tunnel1_details.vpn_ip }}"
            protocol: inet
            myid_type: address
            myid_data: "{{ wan_ip.stdout }}"
            peerid_type: address
            peerid_data: "{{ tunnel1_details.vpn_ip }}"
            preshared_key: "{{ tunnel1_details.shared_secret }}"
            authentication_method: pre_shared_key

          - descr: "t2_OCI_new"
            interface: wan
            iketype: ikev2
            remote_gateway: "{{ tunnel2_details.vpn_ip }}"
            protocol: inet
            myid_type: address
            myid_data: "{{ wan_ip.stdout }}"
            peerid_type: address
            peerid_data: "{{ tunnel2_details.vpn_ip }}"
            preshared_key: "{{ tunnel2_details.shared_secret }}"
            authentication_method: pre_shared_key

    - name: Ensure IPSec Phase2 definitions are updated
      pfsense_ipsec_aggregate:
        aggregated_ipsec_p2s:
          - descr: "t1"
            mode: tunnel
            local: "192.168.0.0/16"
            remote: "10.10.0.0/16"
            protocol: esp
            aes: true
            aes_len: 256
            sha256: true
            pfsgroup: 5
            p1_descr: "t1_OCI_new"

          - descr: "t2"
            mode: tunnel
            local: "192.168.0.0/16"
            remote: "10.10.0.0/16"
            protocol: esp
            aes: true
            aes_len: 256
            sha256: true
            pfsgroup: 5
            p1_descr: "t2_OCI_new"

    - name: IPSec Proposal
      pfsense_ipsec_aggregate:
        aggregated_ipsec_proposals:
          - descr: "t1_OCI_new"
            encryption: aes
            key_length: 256
            hash: sha1
            dhgroup: 5

          - descr: "t2_OCI_new"
            encryption: aes
            key_length: 256
            hash: sha1
            dhgroup: 5
```

For reference, this is the format of the external file ipsec-vars.yml:
```yaml
tunnel1_details:
  shared_secret: goSUhxCj54gwFdJHu1234567890
  vpn_ip: 12.14.14.122
tunnel2_details:
  shared_secret: KlaGWkWy8Wtj78Seu1234567890
  vpn_ip: 10.28.15.177
```
