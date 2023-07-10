- define the file secret-vars.yml with your credentials (for best practices, use ANSIBLE VAULT files encrypted, but for this example a separated file will be enough)

```yml
myapi_user: root@pam
myapi_password: mysupersecretpwd
myapi_host: 192.168.1.100
```

- Ansible file to create a new VM using a pre-defined template - file name proxmox-create.yml

```yml
---
- name: Proxmox stuff
  hosts: srv-proxmox
  tasks:
    - set_fact:
        yeppers: "ubuntu-{{ lookup('password', '/dev/null chars=ascii_lowercase,digits length=8') }}"
    - proxmox_kvm:
        newid: 110
        api_user: "{{ myapi_user }}"
        api_password: "{{ myapi_password }}"
        api_host: "{{ myapi_host }}"
        clone: ubuntu-22.04-template   # The VM source
        name: test-ansible  # The target VM name
        node: homeserver
        storage: local-lvm
        format: qcow2
        description: created with ansible
        timeout: 500  # Note: The task can take a while. Adapt
```

- Ansible file to delete a VM created previously using its name - file name proxmox-delete.yml:

```yml
---
- name: Proxmox stuff
  hosts: srv-proxmox
  tasks:
    - proxmox_kvm:
        api_user: "{{ myapi_user }}"
        api_password: "{{ myapi_password }}"
        api_host: "{{ myapi_host }}"
        name: test-ansible  # The target VM name
        node: homeserver
        state: absent
```

- Commands to run your Ansible files:

```bash
ansible-playbook proxmox-create.yml -i hosts -e @secret-vars.yml

ansible-playbook proxmox-delete.yml -i hosts -e @secret-vars.yml
```

