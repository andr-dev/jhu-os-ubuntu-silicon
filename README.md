# Heyo!

It seems there is quite some interest in getting a linux VM up and running on Apple Silicon with the JHU OS i386 compiler toolchain. Here's the TLDR:

1. Don't use Virtual Box. It's not optimized for Apple Silicon. Use UTM instead.
2. Avoid VM graphics rendering. Loading up another desktop is really resource intensive and graphics acceleration in VMs is quite slow. Use a lightweight window manager for qemu / bochs.
3. Make the SSH server accessible and use VirtFS to create a shared work folder. This allows for your work to be synced between your MacOS filesystem and the VM filesystem and for you to work in VS Code as if it were a ugrad machine (but on your machine so offline and fast).

Do keep in mind that this is a very involved process and will take approximately 2-3 hours to complete so definitely not for the faint hearted!

#

Here's how we'll achieve this.

1. Setup an Ubuntu Server using UTM's x86_64 emulation
1. Make a bridged network interface so that we can SSH into our VM and use VS Code on MacOS when working, and make a shared folder for our work to be synced across both OSes
1. Install a lightweight window manager so we can view qemu / pintos running in our VM for debugging

# Steps

## A) Download and install Ubuntu Server

1. Download and install UTM (https://mac.getutm.app/)

   - Download the app directly from the website, don't pay $9.99 on the app store. The app store version is only there so that if you like the app and want to support their work then you can donate some money. UTM is free and open source. The app store version is not faster or better than the open source one.

1. Download an Ubuntu Server ISO (https://releases.ubuntu.com/22.04.3/ubuntu-22.04.3-live-server-amd64.iso)

1. Open up UTM and create a new emulated virtual machine

   _(I'm on linux and can't open up UTM so these are the rough steps off the top of my head so bear with me)_

   - Click on new virtual machine
   - Select emulate then linux
   - For the ISO, browse to your downloads and select the Ubuntu Server ISO you downloaded
   - Give it a name, doesn't really matter
   - When they ask for disk storage, give it 32 GB ... should be plenty. Once you are done the course, you can delete the entire VM and you'll get all the space back
   - Get to the end of the setup steps :)
     - I don't really remember what else they ask but just do reasonable stuff, if there's ever an architecture question select x86_64 if it's not selected by default

1. Configure the VM

   At this point, you should have a new VM with the name you gave it. Don't run it yet, we need to do some more setup.

   Right click on the box, select edit. Under the System section, set the CPU settings to 4 cores and select "force multicore". Ignore the warning that it's "unstable" lol

1. Install Ubuntu Server

   You can now hit that very tempting play button. This will take some time but eventually you'll get to a setup screen.

   1. Select English
   1. Select Done to keep the default English (US) keyboard variants
   1. Hit done for network connections, no need to change them
   1. Hit done for proxy address, don't want one
   1. Hit done for ubuntu archive mirror, default is fine
   1. Keep use an entire disk, but disable LVM group (use arrow key down then hit space to disable), then click done
   1. At the file system summary, hit done and accept the warning by selecting continue (it's just telling you that if you do this on your actual machine it'll delete all your data, this is in a VM so it won't delete anything)
   1. Pick some names for the profile setup. I would recomend something simple like ubuntuvm for your server's name and jhuos for your name and username. Remember these as you'll need them to SSH into your machine. For password, nobody else will have access to the machine so you can keep it short and sweet
   1. Do not install OpenSSH server, use arrow key down and then done
   1. Don't install any snaps, arrow key down all the way then done

   The machine will now download and install a bunch of software, wait until the Reboot Now menu option pops up and then click it.

   On the next boot, it'll bring you to the GRUB bootloader where you can pick what you want to run. At this point, command Q to close UTM. Open UTM again but do not run the box. Right click the box and hit edit. At the bottom of the left hand side menu, you will see two drives called IDE Drive. One of them will be the ISO that you downloaded (probably the second, maybe the first). Delete the ISO drive, you don't need it anymore. Then select done. Run the VM and after a minute it'll bring you to the login screen. Use your username and password to login. Yay!

   (You can also delete the ISO from your downloads now)

   (QOL: disable cloud-init by running `sudo touch /etc/cloud/cloud-init.disabled`. If you want to completely uninstall it, take a look at https://gist.github.com/zoilomora/f862f76335f5f53644a1b8e55fe98320)

# B) Configure UTM

Now let's change some UTM values! Start by right clicking the VM in UTM and selecting edit (you'll have to shut down the virtual box first, run `sudo shutdown -h now` on the box)

1. Under System, make sure CPU is set to 4 cores and force multi as explained above
1. Under Sharing, make sure VirtFS is selected and then click the browse button. This is the folder that will be shared between your VM and your host machine, so pick the folder under which you want to store your assignments. Do NOT select something like your Desktop or your Downloads folder in its entirety ... one mistaken `rm` command and you'll wipe your Desktop :)
1. Under Display, switch to virtio-gpu-pic (typically better performance)
1. Under Network, switch to Emulated VLAN and select virtio-net-pci. On the left hand side, you'll now see a new section under Network called port forwarding. Open it up and add a new TCP rule that maps guest port 22 to host port 2222.

# C) SSH into VM

At this point your VM should be SSHable. Start the VM and wait until the login page pops up. At this point, open a terminal on your host machine and run `ssh jhuos@localhost -p 2222`. It should warn you that it's the first time you see these keys, type in `y` and hit enter. After typing in your password, you should be in! Let's add an alias for this ssh host in our `~/.ssh/config`.

```
Host jhuos
    Hostname 127.0.0.1
    User jhuos
    Port 2222
```

Now, we can simply run `ssh jhuos` (this will be even more useful in VS Code).

Now, let's open up VS Code. Make sure you've downloaded the Remote SSH extension (https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh). In the remotes tab, since you added the machine to your ssh config, you'll see that it shows up in the remotes list. Nice!

Don't worry, there won't be any latency and you'll still be able to work offline. You'll be communicating with the VM over SSH but it'll be lightning fast since packets are just getting routed through your system's network stack / the virtual network interface we created for the VM.

# C) Install i386 compiler toolchain

Next let's install the important stuff. First, let's make sure all our packages are up to date.

```
sudo apt update
sudo apt upgrade
```

After that's done, we'll start by downloading and extracting the pre-compiled toolchain binaries.

```
wget https://github.com/jhu-cs318/cross-compiler-toolchain/releases/download/v1.0/linux-toolchain-binaries.tar.gz
tar xf linux-toolchain-binaries.tar.gz
rm linux-toolchain-binaries.tar.gz
mkdir -p ~/.local
sudo mv x86_64/* ~/.local
```

To test that our binaries are properly installed, let's try running `which i386-elf-gcc` ...

Oh no :(

Yeah turns out the ubuntu server `~/.bashrc` doesn't include `~/.local/bin` in the PATH by default. Let's fix that by adding the following to the end of `~/.bashrc`.

```
PATH=$HOME/.local/bin:$PATH
```

Quickly run `source ~/.bashrc` and run `which i386-elf-gcc` again and you should now see a binary path. Nice!

Let's also install qemu real quick.

```
sudo apt install qemu
```

Yay! Last but not least, I'm going to sleep so I'll finish this tomorrow ... but wait, how will I know if anyone actually needs this? How about you let me know you desperately need the final steps by staring the repo and giving me a follow :)

(PS: more follow = more happy = more likely I'll finish this guide haha)
