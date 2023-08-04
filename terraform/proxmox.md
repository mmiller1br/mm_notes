
## Terraform and Proxmox

In order to use Terraform to create VMs on my Proxmox server, I've created 3 files (.TF):
- vars.tf (contain variables to be used by other Terraform files)
- provider.tf (contain details for Terraform to connect to the Proxmox provider)
- main.tf (contain details about the resources to be created on Proxmox - in this case, details about the VM to be created)

In order for this automation work, your Proxmox needs to be configured with a user + API_Token, as well as a VM Template to be used. 

Below the files that I used:

### VARS.TF

```
variable "pm_api_url" {
  default = "https://192.168.2.254:8006/api2/json"
}

variable "template_name" {
  default = "ubuntu-22.04-template"
}

variable "proxmox_node" {
  default = "homeserver"
}
```


### PROVIDER.TF

```
terraform {
  required_providers {
    proxmox = {
      source = "telmate/proxmox"
      version = "2.9.14"
    }
  }
}

provider "proxmox" {
  pm_api_url = var.pm_api_url
  pm_api_token_id = "terraform-prov@pve!terraform-tokenid"
  pm_api_token_secret = "ed4da349-1268-4242-b13d-043a053d201a"
  #pm_otp = ""
  pm_tls_insecure = true
  pm_debug = true
}
```


### MAIN.TF

```
resource "proxmox_vm_qemu" "my_tf_test" {
  count = 1
  name  = "my-tf-test-vm"
  desc  = "Test VM created with Terraform"

  target_node = var.proxmox_node

  clone = var.template_name

  agent = 1
  os_type = "cloud-init"
  cores = 1
  sockets = 1
  cpu = "host"
  memory = 2048
  scsihw = "virtio-scsi-pci"
  bootdisk = "scsi0"

  disk {
    slot = 0
    size = "25G"
    type = "scsi"
    storage = "local-lvm"
    iothread = 1
  }

  network {
    model = "virtio"
    bridge = "vmbr0"
  }

}
```


### Terraform Commands

```
terraform init

terraform plan

terraform apply

terraform destroy
```


Happy Automation ! ;)