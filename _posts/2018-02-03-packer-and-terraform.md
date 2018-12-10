---
layout: default
title: "Automating with Packer and TerraForm"
---

[![Po.et verification](https://img.shields.io/badge/Po.et-Verified%20on%20Po.et-green.svg)](link=https://explorer-mainnet.poetnetwork.net/works/64c8a3722898343c19e58004929114983aa1c5fef4fd655eff9c21028d3f9e9d)

### Automating with Packer and TerraForm

[Hashicorp](https://www.hashicorp.com/) have created some amazing tools to help engineers in orchestrating cloud architecture. Two tools I want to go through are [Packer](https://www.packer.io/) and [TerraForm](https://www.terraform.io)

### [Packer.io](https://www.packer.io/)

#### So what exactly is packer
> Packer is an open source tool for creating identical machine images for multiple platforms from a single source configuration. Packer is lightweight, runs on every major operating system, and is highly performant, creating machine images for multiple platforms in parallel. Packer does not replace configuration management like Chef or Puppet. In fact, when building images, Packer is able to use tools like Chef or Puppet to install software onto the image.

> A machine image is a single static unit that contains a pre-configured operating system and installed software which is used to quickly create new running machines. Machine image formats change for each platform. Some examples include AMIs for EC2, VMDK/VMX files for VMware, OVF exports for VirtualBox, etc.

Pre-baked machine images open up a lot in terms of delivery of infrastructure in the modern dev environment. Having the ability to achieve near identical environments running on different providers, can add a world of benefit to any development pipeline. Packer makes this process simple and efficient.

* Super fast infrastructure deployment
* ulti-provider portability
* Improved stability
* Greater testability

### [Terraform](https://www.terraform.io/)
#### So what is TerraForm?

> Terraform is a tool for building, changing, and versioning infrastructure safely and efficiently. Terraform can manage existing and popular service providers as well as custom in-house solutions.

> Configuration files describe to Terraform the components needed to run a single application or your entire datacenter. Terraform generates an execution plan describing what it will do to reach the desired state, and then executes it to build the described infrastructure. As the configuration changes, Terraform is able to determine what changed and create incremental execution plans which can be applied.

### Why use both?

Packer's ability to generate machines images for various builders, means we can create near identical machines across multiple platforms and environments.

TerraFormcan then take the machine image artifacts from Packer, and create the actual resources we need in our infrastructure. This greatly helps us avoid vendor lock-in, as well as ensure that our architecture is provisioned accurately, and quickly, and multiple environments of the architecture can be created seamlessly with minimal effort.

### Infrastructure as Code

Tools such as Packer and TerraForm form part of what is to be knowns as Infrastructure as Code.

> Infrastructure as Code (IaC) is the process of managing and provisioning computing infrastructure (processes, bare-metal servers, virtual servers, etc.) and their configuration through machine-processable definition files

IaC should not be confused with configuration management. IaC is not a substitute, but should implemented in to you orchestration processes along with your configuration management tools.

> Ensure you have the Packer and Terraform binaries installed on your pc.

### Getting Stuck in

A more complete example of Packer and TerraForm can be found [here](https://github.com/WesleyCharlesBlake/terraform-existing-vpc)

In Packer, your definitions are declared in a `.json` template file. I prefer to name my templates based on their purpose (eg: `app-base.json`)

Lets look at the structure of the template (create an empty file `nano myfirst-packer.json` and add the following resources to your template:
```
{
  "builders": [],
  "description": "A packer example template",
  "min_packer_version": "0.8.0",
  "provisioners": [],
  "post-processors": [],
  "variables": []
}
```
Packer needs to be told what to do, where to do it, provision any other boostrapping requirements etc. What does this mean:

`"builders": [],`:
> the builders section contains an array of all the builders that Packer should use to generate a machine images for the template. Builders are responsible for creating machines and generating images from them for various platforms. For example, there are separate builders for EC2, VMware, VirtualBox, etc. Packer comes with many builders by default, and can also be extended to add new builders.

`"provisioners":`
> he provisioners section contains an array of all the provisioners that Packer should use to install and configure software within running machines prior to turning them into machine images.Provisioners are optional. If no provisioners are defined within a template, then no software other than the defaults will be installed within the resulting machine images. This is not typical, however, since much of the value of Packer is to produce multiple identical images of pre-configured software.

`"post-processors":`
> The post-processor section within a template configures any post-processing that will be done to images built by the builders. Examples of post-processing would be compressing files, uploading artifacts, etc.Post-processors are optional. If no post-processors are defined within a template, then no post-processing will be done to the image. The resulting artifact of a build is just the image outputted by the builder.

`"variables":`
> variables allow your templates to be further configured with variables from the command-line, environment variables, or files. This lets you parameterize your templates so that you can keep secret tokens, environment-specific data, and other types of information out of your templates. This maximizes the portability and shareability of the template.

So lets update our template with some builders, provisioners, and variables:
```
{
  "variables": {
      "aws_access_key": "YOURACCESSKEY",
      "aws_secret_key": "YOURSECRETKEY",
      "do_api_token": "YOURAPITOKEN"
  },
  "builders": [{
      "type": "amazon-ebs",
      "access_key": "{{user `aws_access_key`}}",
      "secret_key": "{{user `aws_secret_key`}}",
      "region": "us-east-1",
      "source_ami": "ami-fce3c696",
      "instance_type": "t2.micro",
      "ssh_username": "ubuntu",
      "ami_name": "packer-example {{timestamp}}"
  },{
    "type": "digitalocean",
        "api_token": "{{user `do_api_token`}}",
        "image": "ubuntu-14-04-x64",
        "region": "nyc3",
        "size": "512mb"
  }],
  "provisioners": [{
      "type": "shell",
          "inline": [
              "sleep 30",
              "sudo apt-get update",
              "sudo apt-get install -y redis-server"
    ]
  }]
}
```

The template will not build an AWS AMI, as well as a DigitalOcean snapshot, with the `redis-server` installed. This is very basic, but its upto you to decide how much you want bake into your image, and how much you want to rely on configuration management. This balance is entirely up to your infrastructure/application and deployment requirements.

A more complex provisioner could look like this:
```
"provisioners": [
    {
      "type": "shell",
      "inline": [
        "sudo wget -O - https://repo.saltstack.com/apt/debian/8/amd64/latest/SALTSTACK-GPG-KEY.pub | sudo apt-key add -",
        "sudo su -c 'echo \"deb http://repo.saltstack.com/apt/debian/8/amd64/latest jessie main\" >> /etc/apt/sources.list'",
        "sudo apt-get update",
        "sudo apt-get install -y software-properties-common",
        "sudo apt-get install -y build-essential git",
        "sudo apt-get install -y python python-dev python-setuptools",
        "sudo apt-get install -y salt-minion",
        "sudo apt-get install -y virtualenv",
        "sudo apt-get install -y virtualenvwrapper",
        "sudo wget https://bootstrap.pypa.io/get-pip.py",
        "sudo python get-pip.py",
        "sudo pip install ansible",
      ]
    },
    {
      "type": "salt-masterless",
      "skip_bootstrap": "true",
      "local_state_tree": "/path/to/salt",
      "minion_config" : "/path/to/salt/salt/minion"
    }
    ],
```

This provisioner installs my application build dependencies, installs `salt-minion`, install `virtual-env` and `python pip` and also installs `ansible` (yes, I have a very strange config management strategy, but it works for our use case).

So with these provisioners in the bootstrapping stage, I then can use TerraForm to create the salt-minion ids and keys on creation, and have the salt master register the newly created resource. By baking all the build packages in, I know that each resource created from the image will be concurrent. It helps create a better Dev/Prod parity ("it works on my machine?" excuses ring any bells ;).

To run build your image, packer has some validation tools to help you:

`packer validate myfirst-packer.json`: check that a template is valid (json/syntax errors)

`packer inspect myfirst-packer.json`: see components of a template. This is helpful if you want to see what the output/resources of a template with out having to run a build.

`packer build myfirst-packer.json`: build image(s) from template. If you have multiple builders in your template, the builds will run in parallel (AWESOME!)

`packer fix myfirst-packer.json`: fixes templates from old versions of packer. This is important if you working in a team, and want to ensure the template is the correct version for the version of packer you are running.

### So now that we are building images with Packer, lets use them in TerraForm

Please visit their [site](https://www.terraform.io/downloads.html) and ensure you have TerraForm installed before proceeding.

> Configuration files describe to Terraform the components needed to run a single application or your entire datacenter. Terraform generates an execution plan describing what it will do to reach the desired state, and then executes it to build the described infrastructure. As the configuration changes, Terraform is able to determine what changed and create incremental execution plans which can be applied.

> The infrastructure Terraform can manage includes low-level components such as compute instances, storage, and networking, as well as high-level components such as DNS entries, SaaS features, etc.

For this exmample, we are going to use TerraForm to build infrastructure on AWS.

Lets create a basic TerraForm template: `nano example.tf`:
```
provider "aws" {
  access_key = "ACCESS_KEY_HERE"
  secret_key = "SECRET_KEY_HERE"
  region     = "us-east-1"
}

resource "aws_instance" "example" {
  ami           = "ami-0d729a60" #use ami from packer build
  instance_type = "t2.micro"
}
```

Two most important parts of a TF file are `providers` and `resource` blocks. These essentially tell TerraForm where to build (provider), and what to build (resource). Multiple resource blocks make up your infrastructure (eg you would want a resource block for ec2 resources, another resource block for an elb resource, and so on)

If we are using custom AMIS's (as built by our own packer templates), we would want to use the out put ami id from Packer, in our EC2 resource block in TerraForm. This way TerraForm will use our freshly baked AMI to create the resources. Im sure you can see why this would be handy!

TerraForm's documention is very comprehensive, so go through each resource you are trying to create to get a better understanding.

> Note: You can create a count variable in your resource block, and create n resources from this!

### Plan
Lets plan an execution with TerraForm. In the same directory as your TerraForm template run `terraform plan`. You will see an out put of all the resources that TerraForm would create (without actually creating it!):
```
$ terraform plan
...

+ aws_instance.example
    ami:                      "ami-0d729a60"
    availability_zone:        "<computed>"
    ebs_block_device.#:       "<computed>"
    ephemeral_block_device.#: "<computed>"
    instance_state:           "<computed>"
    instance_type:            "t2.micro"
    key_name:                 "<computed>"
    placement_group:          "<computed>"
    private_dns:              "<computed>"
    private_ip:               "<computed>"
    public_dns:               "<computed>"
    public_ip:                "<computed>"
    root_block_device.#:      "<computed>"
    security_groups.#:        "<computed>"
    source_dest_check:        "true"
    subnet_id:                "<computed>"
    tenancy:                  "<computed>"
    vpc_security_group_ids.#: "<computed>"
```

If you are happy with out put, you can run plan, alternatively you may need to fix any errors etc. You might also want to store the out put of plan, in order run the execution. There are benefits of storing the plan output

### Apply
Run `terraform apply` in the same directory as your example.tf to create the resources!

You should see something like this:
```
$ terraform apply
aws_instance.example: Creating...
  ami:                      "" => "ami-0d729a60"
  instance_type:            "" => "t2.micro"
  [...]

aws_instance.example: Still creating... (10s elapsed)
aws_instance.example: Creation complete

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

...
```

TerraForm is busy creating your resources!


### State

An important concept to be aware of is state.

Terraform also put some state into the terraform.tfstate file by default. This state file is extremely important; it maps various resource metadata to actual resource IDs so that Terraform knows what it is managing. This file must be saved and distributed to anyone who might run Terraform. We recommend simply putting it into version control, since it generally isn't too large.

You can inspect the state using `terraform show`:
```
$ terraform show
aws_instance.example:
  id = i-32cf65a8
  ami = ami-0d729a60
  availability_zone = us-east-1a
  instance_state = running
  instance_type = t2.micro
  private_ip = 172.31.30.244
  public_dns = ec2-52-90-212-55.compute-1.amazonaws.com
  public_ip = 52.90.212.55
  subnet_id = subnet-1497024d
  vpc_security_group_ids.# = 1
  vpc_security_group_ids.3348721628 = sg-67652003
```

To destroy resources, you could run `terraform destroy` and it will handle the rest.

> Note: be aware of resources changes that require the resource to be destroyed and recreated. Changing an AWS EC2 resource AMI to something new, means the old EC2 images will be destroyed and recreated. Besure to plan these changes correctly, or you could have a very bad day.

Thats it for now! If you have any questions give me a shout!

I hope this helps you understand Packer and TerraForm a bit better.
