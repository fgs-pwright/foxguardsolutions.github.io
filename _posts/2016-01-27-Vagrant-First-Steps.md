---
layout: post
title: Vagrant First Steps - Hyper-V Host and Windows Server Core Guest
image: /images/2016-01-27-Vagrant-First-Steps/Vagrant.png
author: Bruce Markham
excerpt: Ultimately looking to automate test environment & build slave spin-up, we had a few constraints that made things complicated. We ran into snags, so keep reading if you plan on dipping your toes in this water.
---
Attempting to broaden our horizons a little bit, a few of us settled down at [CoP](https://en.wikipedia.org/wiki/Community_of_practice "Wikipedia: Community of Practice") the other day to play with [Vagrant](https://www.vagrantup.com/).

Ultimately looking to automate test environment & build slave spin-up, we had a few constraints that made things complicated:

  - We needed to run Windows Server Core guest OSes, (we use a lot of the Microsoft stack, and Server Core is small and made for remote/console config.)
  - We needed to use Hyper-V for hosting, (we have Hyper-V resources in-house to leverage.)
  - We wanted to use [Chef](https://www.chef.io/chef/) (because it seems like that's what people "do".)

After trying Vagrant `box` after `box`, we met little success finding a pre-built combination of the above three that either worked or was from a trustworthy source. It felt to me like we were trying to attack this technology stack from the middle, so we decided to spend some time on a different approach: building a Windows `box` from scratch. We ran into even more snags, so keep reading if you plan on dipping your toes in this water.

The official documentation is a good place to start, albeit terse in the practicality department:

  - [Vagrant: Hyper-V: Creating A Base Box](https://docs.vagrantup.com/v2/hyperv/boxes.html)
  - which leads you to the slightly more generalized [Vagrant: Creating A Base Box](https://docs.vagrantup.com/v2/boxes/base.html)

From my novice understanding, if we wanted to do this "fer srs", we should probably use something like [Packer](https://www.packer.io/intro/index.html), along with a toolkit+guide like [this one](https://github.com/joefitzgerald/packer-windows "GitHub: joefitzgerald/packer-windows"). However, this doesn't address our need to learn about what goes "into" a `box`, and a small [scavenger](https://github.com/MSOpenTech/packer-hyperv/issues/18 "defunct Packer + Hyper-V effort") [hunt](https://github.com/pbolduc/packer-hyperv/ "another defunct Packer + Hyper-V effort") shows that Packer + Hyper-V don't play well together [yet](https://github.com/mitchellh/packer/pull/2576 "terrifyingly large pull request to Packer, adding Hyper-V support"). So we set out to do it the old-fashioned way.

### Here Are The Steps We Followed

1. We needed a Windows Server ISO, so we fired up our MSDN subscription and downloaded `en_windows_server_2016_technical_preview_4_x64_dvd_7258292.iso` because we were feeling adventurous.

2. We used Hyper-V Manager to create a `Generation 2` Virtual Machine named `WinServ2016CTP4Core`, with `2048` MB of (dynamic) startup memory, attached to an `External` virtual switch, with a new VHDX with the same name as the VM.

3. We mounted the ISO to the VM, fired it up, and installed Windows Server 2016 Technical Preview 4 in `Core` mode.

4. We gave the `Administrator` account a random (complex) password. (You can change it later with `net user`, as [this post](http://www.thewindowsclub.com/net-user-command-windows) shows.)

5. "Turn off UAC" - no need, it is already disabled in Server Core.

6. "Disable complex passwords" - [this post](http://servercore.net/index.php/2014/01/how-to-disable-password-complexity-on-server-core-installations) shows how to use `secedit` to disable password complexity.

7. "Disable 'Shutdown Tracker'" - this step didn't seem necessary because our Server Core wasn't questioning our shutdowns, but if you are dying to check it out, [this is how you do it](https://blogs.technet.microsoft.com/chenley/2011/03/05/how-to-disable-the-shutdown-tracker-in-windows-server-2008-r2/) with `gpedit.msc`, and [this is a good starting point](http://blogs.msdn.com/b/neilhut/archive/2007/11/06/managing-local-policy-on-a-windows-server-core-installation-set-to-workgroup-config.aspx) for using `gpedit.msc` against a remote (Server Core) machine.

8. "Disable 'Server Manager' starting at login (for non-Core)" - self-explanatory, nothing to do here.

9. "Base WinRM Configuration" - Vagrant's doc's formatting is a bit lacking here, so let us break it down for you:
  - `winrm quickconfig -q`
  - `winrm set winrm/config/winrs @{MaxMemoryPerShellMB="512"}`
  - `winrm set winrm/config @{MaxTimeoutms="1800000"}`
  - `winrm set winrm/config/service @{AllowUnencrypted="true"}`
  - `winrm set winrm/config/service/auth @{Basic="true"}`
  - `sc config WinRM start= auto`
  
  NOTE: We'll repeat a couple of Vagrant's pieces of advice here:
    - Use the "Windows command prompt" for the above - _not_ _PowerShell_.
    - The above steps purposefully create some security holes - make sure you plug them before going to production.

10. We were curious how to install Windows Updates on the server, and found [this beauty](http://blog.zwiegnet.com/windows-server/install-windows-updates-on-server-2012-core/). Apparently `sconfig` is totally a thing.

11. We then used `sconfig` to `Add Local Administrator` with username `vagrant` and password `vagrant`, which makes Vagrant's scripts happy.

12. If you are following along and want to install `Chocolatey`, `Chef`, `Puppet`, etc. you could probably do that now. (We haven't tried yet.)

13. Then we shut down the VM and told Hyper-V Manager to export it.

14. Then we found the VHDX that is part of the exported VM in Windows Explorer, mounted it, defragged it, and then unmounted it again. This makes the following step more effective.

15. So then we used Hyper-V Manager's `Edit Virtual Hard Disk Wizard` to `Compact` the disk. This and the above step make sure the VHDX is as small as we can get it without a thorough cleanup of the virtual drive.

16. Next we edited the `XML` file in the `Virtual Machines` folder from the VM export, to make sure it uses a relative path to the VHDX of the export. This is important if you plan to move things around.

17. Next we deleted the `Snapshots` folder from the VM export folder (it was a sibling to the `Virtual Machines` and `Virtual Hard Disks` folders.)

18. In that same folder, as the Vagrant docs suggest, we created a `metadata.json` file that looked like this:

	```json
 	{
		"provider": "hyperv"
	}
	```

19. Next we took everything in that folder, and made a `tar` with 7-zip, named `WinServ2016CTP4Core-hyperv.tar`.

	NOTE: Vagrant docs indicate you can make and use a `Zip`, but this didn't work out for us - the `bsdtar` Vagrant has baked-in will fail to unpack the files in the next step if you use a Zip. we also tried to do a 7-zip "Ultra" Zip to make the box as our original step. This took a couple of hours, and only reduced the overall size by 50% (vs no compression). And then the Zip wasn't usable. Don't make these mistakes.
	
20. Then, from that folder, we ran:

	`vagrant box add ./WinServ2016CTP4Core-hyperv.tar --name bmarkham/WinServ2016CTP4Core --provider hyperv`

21. And voila! Then we could create a simple `Vagrantfile` like this:

	```ruby
	Vagrant.configure(2) do |config|
	  config.vm.box = "bmarkham/WinServ2016CTP4Core"
	  config.vm.communicator = :winrm
	  config.vm.provider "hyperv" do |hv|
	    	hv.vmname = "WinServ2016CTP4Core"
	    	hv.cpus = 2
	    	hv.memory = 2048
		end
	end
	```

    ...for which a `vagrant up` works just fine.
	
    NOTE: Neglecting to specify the `winrm` communicator will cause extra delay followed by a failure at the end of the `vagrant up` process.
	
    Also NOTE: After all this, we made the mistake of deleting the resulting Hyper-V VM through Hyper-V Manager (instead of `vagrant destroy`.) When we re-ran `vagrant up`, it crashed. Looks like [we found a bug](https://github.com/mitchellh/vagrant/issues/6882) (and a work-around).

### Conclusion

While we're sure `Chef` and `Packer` (sans Hyper-V) make the common case of this easier, we honestly had to experience all of this before attempting to build on it. (Some of these more optional "experiences" consumed several hours in total.)

Hope this helps you folks, too.
