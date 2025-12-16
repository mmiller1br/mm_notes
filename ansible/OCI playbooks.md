playbooks here in this document:
1. delete S2SVPN config and CPE created from your OCI instance
2. create CPE and S2SVPN on your OCI instance
xx-xx-xx-xx

- 1) Delete existing S2SVPN and CPE previously created.
```yaml
---
- name: Delete existing OCI S2S VPN and its associated CPE
  hosts: localhost
  connection: local
  gather_facts: no

  vars:
    # Compartment where the VPN and CPE exist
    compartment_id: "ocid1.tenancy.oc1.your_instanc_id_here"

    # Display names to locate the objects
    vpn_name: "VPN-vcn_myhomelab"
    cpe_name: "my-pfsense"

    # OCI authentication (using your ~/.oci/config)
    oci_config_file: "~/.oci/config"
    oci_config_profile: "DEFAULT"

  tasks:

    ##################################################################
    # Step 1: Locate the IPSec VPN by display name
    ##################################################################
    - name: Get existing IPSec connections
      oracle.oci.oci_network_ip_sec_connection_facts:
        compartment_id: "{{ compartment_id }}"
        config_file_location: "{{ oci_config_file }}"
        config_profile_name: "{{ oci_config_profile }}"
      register: ipsec_list
      #ignore_errors: yes          # <-- add this temporarily

    - name: Filter IPSec connection by name
      set_fact:
        ipsec_to_delete: >
          {{ ipsec_list.ip_sec_connections
             | selectattr('display_name','equalto', vpn_name)
             | list
             | first
             | default(None)
          }}

    - name: Show ipsec_to_delete
      debug:
        var: ipsec_to_delete

    - name: Show found IPSec ID
      debug:
        msg: "Found IPSec connection: {{ ipsec_to_delete.display_name | default('N/A') }} | ID = {{ ipsec_to_delete.id | default('N/A') }}"
      when: (ipsec_to_delete is not none) and (ipsec_to_delete != "\n")

    - name: No tunnel found
      debug:
        msg: "No IPSec connection named '{{ vpn_name }}' exists – nothing to do"
      when: (ipsec_to_delete is none) or (ipsec_to_delete == "\n")


    ##################################################################
    # Step 2: Delete the IPSec connection
    ##################################################################
    - name: Delete IPSec connection
      oracle.oci.oci_network_ip_sec_connection:
        ipsc_id: "{{ ipsec_to_delete.id }}"
        state: absent
        config_file_location: "{{ oci_config_file }}"
        config_profile_name: "{{ oci_config_profile }}"
      when: (ipsec_to_delete is not none) and (ipsec_to_delete != "\n")


    ##################################################################
    # Step 3: Wait for IPSec deletion to complete
    ##################################################################
    - name: Wait until IPSec connection is fully removed
      oracle.oci.oci_network_ip_sec_connection_facts:
        compartment_id: "{{ compartment_id }}"
        config_file_location: "{{ oci_config_file }}"
        config_profile_name: "{{ oci_config_profile }}"
      register: ipsec_check
      until: >
        ipsec_check.ip_sec_connections
        | selectattr('display_name','equalto', vpn_name)
        | list
        | length == 0
      retries: 20
      delay: 10
      when: (ipsec_to_delete is not none) and (ipsec_to_delete != "\n")


    ##################################################################
    # Step 4: Find the CPE object by name
    ##################################################################
    - name: Get CPE list
      oracle.oci.oci_network_cpe_facts:
        compartment_id: "{{ compartment_id }}"
        config_file_location: "{{ oci_config_file }}"
        config_profile_name: "{{ oci_config_profile }}"
      register: cpe_list

    - name: Filter CPE by name
      set_fact:
        cpe_to_delete: >
          {{ cpe_list.cpes
             | selectattr('display_name','equalto', cpe_name)
             | first
             | default(None) }}

    - name: Show found CPE ID
      debug:
        var: cpe_to_delete


    ##################################################################
    # Step 5: Delete the CPE
    ##################################################################
    - name: Delete CPE
      oracle.oci.oci_network_cpe:
        cpe_id: "{{ cpe_to_delete.id }}"
        state: absent
        config_file_location: "{{ oci_config_file }}"
        config_profile_name: "{{ oci_config_profile }}"
      when: (cpe_to_delete is not none) and (cpe_to_delete != "\n")


    ##################################################################
    # Step 6: Wait for CPE deletion to complete
    ##################################################################
    - name: Wait until CPE is fully removed
      oracle.oci.oci_network_cpe_facts:
        compartment_id: "{{ compartment_id }}"
        config_file_location: "{{ oci_config_file }}"
        config_profile_name: "{{ oci_config_profile }}"
      register: cpe_check
      until: >
        cpe_check.cpes
        | selectattr('display_name','equalto', cpe_name)
        | list
        | length == 0
      retries: 20
      delay: 10
      when: (cpe_to_delete is not none) and (cpe_to_delete != "\n")


    ##################################################################
    # Final Status
    ##################################################################
    - name: Final summary
      debug:
        msg: >
          VPN '{{ vpn_name }}' and CPE '{{ cpe_name }}'
          have been deleted (or already did not exist).
```


- 2) Create CPE and S2SVPN config on OCI - Oracle Cloud Infrastructure
As the IPSec tunnel will be terminated i a PFSense firewall, we need at least the WAN interface IP to create the CPE config on OCI. This information is in an external file pfsense-vars.yml.
At the end, save the information about tunnel1 and tunnel2 in another external file to be used during the PFSense reconfiguration - see playbooks about PFsense IPSec.

```yaml
---
- name: Create CPE and IPSec VPN
  hosts: localhost
  connection: local
  gather_facts: no
  vars_files:
    - ../pfsense/pfsense-vars.yml

  vars:
    cpe_ip: "{{ wan_ip }}"       # directly available as wan_ip
    cpe_name: "my-pfsense"
    ipsec_name: "VPN-vcn_myhomelab"

    compartment_ocid: "ocid1.tenancy.oc1._your_oci_id_here_"
    drg_ocid: "ocid1.drg.oc1.ca-toronto-1._drg_id_here_"

    vcn_name: "vcn_myhomelab"
    oci_config_file: "~/.oci/config"
    oci_profile: "DEFAULT"

    # Static routes for OCI → On-Prem
    onprem_static_routes:
      - "192.168.5.0/24"

    # Local OCI subnets that should be reachable from on-prem
    local_vcn_subnets:
      - "10.10.10.0/24"
      - "10.10.20.0/24"

  tasks:

    ##################################################################
    # 1) Look up VCN by name
    ##################################################################
    - name: Get VCN list in compartment
      oracle.oci.oci_network_vcn_facts:
        config_file_location: "{{ oci_config_file }}"
        config_profile_name: "{{ oci_profile }}"
        compartment_id: "{{ compartment_ocid }}"
      register: vcn_list

    - name: Extract VCN OCID
      set_fact:
        vcn_ocid: "{{ item.id }}"
      loop: "{{ vcn_list.vcns }}"
      when: item.display_name == vcn_name

    - name: Fail if VCN not found
      fail:
        msg: "VCN '{{ vcn_name }}' not found in compartment!"
      when: vcn_ocid is not defined


    ##################################################################
    # 2) Create/Ensure CPE exists
    ##################################################################
    - name: Create CPE object
      oracle.oci.oci_network_cpe:
        config_file_location: "{{ oci_config_file }}"
        config_profile_name: "{{ oci_profile }}"
        compartment_id: "{{ compartment_ocid }}"
        display_name: "{{ cpe_name }}"
        ip_address: "{{ cpe_ip }}"
        state: present
      register: cpe_result

    - debug:
        msg: "CPE created/exists: {{ cpe_result.cpe.id }}"


    ##################################################################
    # 3) Create IPSec Connection
    ##################################################################
    - name: Create IPSec Connection
      oracle.oci.oci_network_ip_sec_connection:
        config_file_location: "{{ oci_config_file }}"
        config_profile_name: "{{ oci_profile }}"
        compartment_id: "{{ compartment_ocid }}"
        cpe_id: "{{ cpe_result.cpe.id }}"
        drg_id: "{{ drg_ocid }}"
        display_name: "{{ ipsec_name }}"
        static_routes: "{{ onprem_static_routes }}"
        tunnel_configuration:
          - routing: STATIC
            ike_version: V2
          - routing: STATIC
            ike_version: V2
        state: present
        wait: yes
        wait_timeout: 600
      register: ipsec_result


    ##################################################################
    # 4) Retrieve tunnel list for the IPSec connection
    ##################################################################
    - name: Get IPSec tunnel list
      oracle.oci.oci_network_ip_sec_connection_tunnel_facts:
        config_file_location: "{{ oci_config_file }}"
        config_profile_name: "{{ oci_profile }}"
        ipsc_id: "{{ ipsec_result.ip_sec_connection.id }}"
      register: tunnel_facts


    ##################################################################
    # 5) Retrieve tunnel IPs + SHARED SECRETS (final working version)
    ##################################################################
    - name: Get IPSec connection by display name
      oracle.oci.oci_network_ip_sec_connection_facts:
        compartment_id: "{{ compartment_ocid }}"
        display_name: "{{ ipsec_name }}"
      register: ipsec_conn

    - name: Fail if not found
      ansible.builtin.fail:
        msg: "IPSec connection '{{ ipsec_name }}' not found!"
      when: ipsec_conn.ip_sec_connections | length == 0

    - name: Set IPSec connection OCID
      ansible.builtin.set_fact:
        ipsec_ocid: "{{ ipsec_conn.ip_sec_connections[0].id }}"

    # This is the ONLY module that returns the shared_secret
    - name: Get IPSec tunnel configuration (includes shared secrets!)
      oracle.oci.oci_network_ip_sec_connection_device_config_facts:
        ipsc_id: "{{ ipsec_ocid }}"
      register: device_config

    - name: Extract tunnel IPs and shared secrets
      ansible.builtin.set_fact:
        tunnel1_ip:     "{{ device_config.ip_sec_connection_device_config.tunnels[0].ip_address }}"
        tunnel1_secret: "{{ device_config.ip_sec_connection_device_config.tunnels[0].shared_secret }}"
        tunnel2_ip:     "{{ device_config.ip_sec_connection_device_config.tunnels[1].ip_address }}"
        tunnel2_secret: "{{ device_config.ip_sec_connection_device_config.tunnels[1].shared_secret }}"

    - name: Show tunnel details (with secrets!)
      ansible.builtin.debug:
        msg: |
          Tunnel 1:
            Oracle Public IP : {{ tunnel1_ip }}
            Shared Secret    : {{ tunnel1_secret }}
          Tunnel 2:
            Oracle Public IP : {{ tunnel2_ip }}
            Shared Secret    : {{ tunnel2_secret }}

    - name: Build the desired ipsec-vars dictionary
      ansible.builtin.set_fact:
        ipsec_vars:
          tunnel1_details:
            vpn_ip: "{{ device_config.ip_sec_connection_device_config.tunnels[0].ip_address }}"
            shared_secret: "{{ device_config.ip_sec_connection_device_config.tunnels[0].shared_secret }}"
          tunnel2_details:
            vpn_ip: "{{ device_config.ip_sec_connection_device_config.tunnels[1].ip_address }}"
            shared_secret: "{{ device_config.ip_sec_connection_device_config.tunnels[1].shared_secret }}"

    - name: Write ipsec-vars.yml in the same folder as the playbook
      ansible.builtin.copy:
        content: "{{ ipsec_vars | to_nice_yaml(indent=2) }}"
        dest: "{{ playbook_dir }}/ipsec-vars.yml"
        mode: '0644'
      delegate_to: localhost
      run_once: true
```

Format of the file ipsec-vars.yml created after tun this playbook:
```
tunnel1_details:
  shared_secret: goSUhxC54gwFdJHunAx1234567890
  vpn_ip: 12.14.14.29
tunnel2_details:
  shared_secret: KlaGWkWytj78SeHz6ru1234567890
  vpn_ip: 10.28.11.11
```

