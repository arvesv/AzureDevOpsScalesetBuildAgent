# How I configure Azure Devops ScaleSet based build agents

## Background
Azure DevOps Pipelines have several options for build agents. The easiest is to use the Microsoft standard hosted agents (like ubuntu-latest or windows-latest) but
sometimes there are cases where these does not fit. You may need a larger disk, more CPU or RAM or other special software that is not included in the standard agents.
Or you may require the the build agent should be in your local network.  The standard Micorosft hosted agents has a fixed monthly price based on number of agents you allocate. 

You could use selfhosted agents, and there are use cases for that. But scaling is next to impossible.

The third option is using Azure ScaleSet based agents. This means creating an Azure VM image, and connect the Azure Agent pool to a ScaleSet based on VM image. The advantages are:
*  It scales the number of Agents based on jobs you submits - It is "pay per use" not "pay per month" so the cost will usually be lower
*  You can get right machine type for your compilation needs as you need as you choose the machine size (CPU/RAM) and disk type and size

The "pay per use" model (dare we call it a "serverless" pricing model?) works well both in an enteprice where the need for agents varies througout the day. It also works for a solo developer that occationately needs run a job that does not fit on the standard agent.

The disadvantages of using the Azure ScaleSet based agents.
* You have to maintain the image yourself - i.e updating the software and operating system at regular intervals.
* The cost of storing the VM image - but this is almost nothing if you use the "Standard HHD" performance tier for storing the image.

To create an Azure DevOps ScaleSet based agent pool you have to do:
1. Create a Virtual Machine
2. Install the software
3. "Generalize" the VM to create an image
4. Create the ScaleSet and connect it to Azure DevOps.

And at regular intervals you have to update the image, and that means:
1. Create a new VM based on the latest version of the image
2. Update the operating system/installed software on the VM
3. "Generalize" the VM to create an image
4. Update the ScaleSet to use the new image

The process is described more in detail below.

Disclaimer: I have mostly worked with Windows based agents, Linux agents may be different.


## Creating the Virtual Machine

Before you create machine, you have to think about a about few questions.  

*Do you want to use ephemeral disks for you ScaleSet?* The answer is probably yes, if you can. An ephemeral disk is a locally attached SSD on the VM, used for the operating system disk. It is quick, does not cost anything and is deleted if the machine is rebooted - a perfect match for a build agent. But the different Azure VMs sizes have different maximum sizes for ephemeral disks, and they are usually quite small or zero. If you have a partiular VM size in mind, make sure that the boot disk it not larger than the ephemeral disk size. This means that you may have to use the "Small Disk" versions when creating the VM, and then increase the disk size to just below the maximum ephemeral disk size. 

*Where to put the workspace? On the temporary disk or somewhere else?* This is something that can be changed at a later stage, but you should have thought about where the workspace should be located. The temprary disk is usually a good idea, but it needs enough space.

There are also a few other question like  VM size, Gen 1 og Gen 2 

You also need to answer all the "normal" questions when creating a VM: size, generation, network and more.



## Installing the software

In the future when/if I figure it out, I may probably talk about Ansible, DSC or other ways to install the software.   But a normal manual install works, as long as you install the tools for all users. The current user will be deleted when generalizing the VM.

But it is a good idea to script as much as possible. But old stuff om Windows is not always scripable.

If I have a particular build in mind, I usually test the machine but manually compiling something. 

Interesting links (at least on the Windows platform):
* [Software installed on the Microsoft agent](https://github.com/actions/virtual-environments/blob/main/images/win/Windows2019-Readme.md)
* [Chocolayey package manager for Windows](https://chocolatey.org/)

When everything is OK, I reboot the VM 

## "Generalize" the VM to create an image

This is very Windows specific. I don't know if the same concept exists on Linux.

You need to run c:\Windows\System32\sysprep\sysprep.exe

![Sysprep image](docs/sysprep.png)

I have only used "generalize" and "shutdown". I think I have read that the machine will be unusable after this operation, but I have never chcked if that is actually true.

