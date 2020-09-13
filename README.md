# Instruction - Packer Image Deployment

### Pre-requisites
  
1. <b>Packer Installation:</b></br>
    Install packer and configure appropriate binary path following the below link.</br>

      <a href="https://learn.hashicorp.com/tutorials/packer/getting-started-install">https://learn.hashicorp.com/tutorials/packer/getting-started-install</a></br>

2. <b>Configuring AWS Profile:</b></br>
    Follow the below steps to create aws default profile. The user must have the required permissions to create resources.</br>

        $ aws configure
        AWS Access Key ID [None]: {Access Key}
        AWS Secret Access Key [None]: {Secret Access Key}
        Default region name [None]: {Leave as blank}
        Default output format [None]: {Leave as blank as default is JSON}

### Packer Templating
  
1. Variable Definition</br>
    Any frequently changing attributes between each build can be configured in variables section.</br></br>
 ```json
"variables": {
		"aws_access_key": "",
		"aws_secret_key": "",
		"region": "us-east-1",
		"version": "1.0"
}
```

2. Builder Section</br>
    Builders are responsible for creating machines and generating images from them for various platforms. For example, there are separate builders for EC2, VMware, VirtualBox, etc.    Packer comes with many builders by default and can also be extended to add new builders.</br>
    Generally, to create the AMIs in AWS we use EBS backed AMI builder.</br></br>
```json
"builders": [{
		"type": "amazon-ebs",
		"access_key": "{{user `aws_access_key`}}",
		"secret_key": "{{user `aws_secret_key`}}",
		"region": "{{user `region`}}",
		"source_ami_filter": {
		  "filters": {
			"name": "centos-7-base*"
		  },
		  "owners": "161831738826",
		  "most_recent": true
		},
		"ami_name": "mtapp-{{user `version`}}-{{timestamp}}",
		"instance_type": "t2.micro",
		"ssh_username": "centos",
		"tags": {
			"version": "mtapp-{{user `version`}}",
			"base": "{{ .SourceAMIName }}"
		}
	}]
```

3. Provisioners</br>
    Provisioners use built-in and third-party software to install and configure the machine image after booting. Provisioners prepare the system for use.</br></br>
```json
"provisioners": [
		{
			"type": "shell",
			"inline": [
				"echo 'Updating the system with latest patch'",
				"sudo yum update -y",
				"echo 'Creating centos User'",
				"sudo usermod -aG wheel centos",
				"sudo sed -i 's/^%wheel/# %wheel/' /etc/sudoers",
				"echo '%wheel  ALL=(ALL)       NOPASSWD: ALL' | sudo tee -a /etc/sudoers",
				"echo centos:centos | sudo chpasswd",
				"echo 'Installing Ansible'",
				"sudo yum install -y epel-release",
				"sudo yum install -y ansible",
				"echo 'Installing python & pip'",
				"sudo yum install -y python python-pip python3 python3-pip"
			]
		},
		{
			"type": "file",
			"source": "docker/docker-compose.yml",
			"destination": "/home/centos/docker-compose.yml"
		},
		{
			"type": "ansible-local",
			"playbook_file": "docker/mtapp.yml"
		},
		{
			"type": "shell",
			"inline": [
				"echo 'Removing Ansible'",
				"sudo yum remove -y ansible"
			]
		}
	]
```

<b>Shell Provisioner</b></br>
The shell Packer provisioner provisions machines built by Packer using shell scripts. Shell provisioning is the easiest way to get software installed and configured on a machine.</br></br>
 
<b>Ansible Local Provisioner</b></br>
The ansible-local Packer provisioner will run ansible in ansible's "local" mode on the remote/guest VM using Playbook and Role files that exist on the guest VM. This means ansible must be installed on the remote/guest VM. </br></br>
Playbooks and Roles can be uploaded from your build machine (the one running Packer) to the VM. Ansible is then run on the guest machine in local mode via the ansible-playbook command.</br></br>
 
<b>Ansible Remote Provisioner</b></br>
The ansible Packer provisioner runs Ansible playbooks. It dynamically creates an Ansible inventory file configured to use SSH, runs an SSH server, executes ansible-playbook, and marshals Ansible plays through the SSH server to the machine being provisioned by Packer.</br></br>
 
<b>File Provisioner</b></br>
The file Packer provisioner uploads files to machines built by Packer. The recommended usage of the file provisioner is to use it to upload files, and then use shell provisioner to move them to the proper place, set permissions, etc.</br></br>

### Image Build Operations

1. Packer Validate</br>
    Execute below command to validate the template.</br>

        $ packer validate mtapp-amibuild.json

2. Packer Build</br>
    To build a customer AMI which has the mtapp application, execute the below command. This will result the AMI ID at the end of successful execution</br>

        $ packer build mtapp-amibuild.json
    
    To use different authentication options please refer the below link.

      <a href="https://www.packer.io/docs/builders/amazon#specifying-amazon-credentials">https://www.packer.io/docs/builders/amazon#specifying-amazon-credentials</a></br>

