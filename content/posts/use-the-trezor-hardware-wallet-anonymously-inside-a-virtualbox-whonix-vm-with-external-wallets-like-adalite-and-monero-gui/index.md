---
title: "Use the Trezor Hardware Wallet Anonymously Inside a VirtualBox Whonix VM With External Wallets Like Adalite and Monero GUI"
date: 2021-10-22T20:15:36+02:00
cover: "img/cover.png"
useRelativeCover: true
hideReadMore: true
comments: true
tags:
- adalite
- anonymity
- cardano
- cryptocurrency
- monero
- privacy
- qubes os
- trezor
- virtualbox
- whonix
---

In the past, I used an old laptop running [Qubes OS](https://www.qubes-os.org/) for any cryptocurrency-related stuff, and it worked great. It's where I first learned about [Whonix](https://www.whonix.org), a desktop operating system designed to protect your privacy online. Unfortunately, Qubes OS is a bit picky about the hardware it runs on. My old laptop only has four gigs of RAM, and I could barely run two instances of [MyEtherWallet](https://www.myetherwallet.com) in two separate [qubes](https://www.qubes-os.org/doc/glossary/#qube) without the system running out of memory.

## The Final Straw

When initially configuring my Qubes OS environment for Monero, I decided to go with [the setup described in a user guide from the official Monero website](https://www.getmonero.org/resources/user-guides/cli_wallet_daemon_isolation_qubes_whonix.html):

> With Qubes + Whonix you can have a Monero wallet that is without networking and running on a virtually isolated system from the Monero daemon which has all of its traffic forced over Tor.

Out of the blue, after more than a year of using this setup, I couldn't start the daemon and the wallet GUI simultaneously anymore. I only had enough RAM to launch one, but not both, which made the setup unusable. So I had to buy new hardware.

## Reducing The Complexity

I would have loved to get a [NitroPad X230](https://shop.nitrokey.com/shop/product/nitropad-x230-67), a [Qubes OS-certified](https://www.qubes-os.org/doc/certified-hardware/) laptop tinker around with Qubes OS some more. Also, the guys at [Nitrokey](https://www.nitrokey.com/) are doing a great job in the open-source space and deserve any support they get. But I ultimately decided to reduce the complexity of my crypto setup and went with a hardware wallet instead — the [Trezor Model T](https://shop.trezor.io/product/trezor-model-t).

I'll guide you through my new "Whonix on VirtualBox" setup and the steps required to configure it.

## Import the VirtualBox Whonix Appliance

First, [we download the VirtualBox appliance provided by Whonix](https://www.whonix.org/wiki/VirtualBox/XFCE) and [make sure to verify the binary](https://www.whonix.org/wiki/VirtualBox/Verify_the_virtual_machine_images_using_the_command_line). This requires different steps depending on the OS you are using.

Next, we import the appliance into VirtualBox by clicking **File** &rarr; **Import Appliance...**, creating two VMs — the gateway and the workstation. The gateway connects us to the Tor network. The workstation will only connect through that gateway, so both VMs must run if we want to connect to the outside world from within the workstation VM.

Make sure to read the [Post-installation Security Advice from the official Whonix docs](https://www.whonix.org/wiki/Post_Install_Advice#Security_Updates). We [change the default user password](https://www.whonix.org/wiki/Post_Install_Advice#Change_Password) and [install the latest updates](https://www.whonix.org/wiki/Operating_System_Software_and_Updates) for the gateway at the very least. After, we repeat the same for the workstation VM.

At this point, we can optionally clone the workstation VM to have a clean Whonix image if we ever want to start over from scratch.

## Install the Trezor Dependencies

We can now connect the Trezor device to our computer. We then right-click on the workstation VM and select **Settings...**. Under **USB**, we check the **Enable USB Controller** and add the **Trezor** device. This way, it will be automatically attached to the workstation VM.

![Enable USB Controller](img/enable-usb-controller.png)

Let's start the VM, open a terminal, and [import the udev rule for Trezor](https://wiki.trezor.io/Udev_rules) to enable communication with the Trezor device via Linux kernel:

```shell
sudo curl https://data.trezor.io/udev/51-trezor.rules -o /etc/udev/rules.d/51-trezor.rules
```

Next, we [download the SatoshiLabs 2021 Signing Key](https://trezor.io/security/satoshilabs-2021-signing-key.asc):

```shell
wget https://trezor.io/security/satoshilabs-2021-signing-key.asc
```

We verify the authenticity of the key by running:

```shell
gpg --with-fingerprint ./satoshilabs-2021-signing-key.asc
```

The value of the fingerprint is `EB48 3B26 B078 A4AA 1B6F  425E E21B 6950 A2EC B65C`. If it's not, something is very wrong — do **NOT** continue! If everything looks good, we import the key:

```shell
gpg --import ./satoshilabs-2021-signing-key.asc
```

Then we download the [Trezor Suite](https://suite.trezor.io/) and corresponding signature by running the following in the terminal (replace `XX.XX.X` with the latest version):

```shell
wget https://suite.trezor.io/web/static/desktop/Trezor-Suite-XX.XX.X-linux-x86_64.AppImage
wget https://suite.trezor.io/web/static/desktop/Trezor-Suite-XX.XX.X-linux-x86_64.AppImage.asc
```

We verify the binaries by running:

```shell
gpg --verify ./Trezor-Suite-XX.XX.X-linux-x86_64.AppImage.asc
```

The output should contain:

```text
Good signature from "SatoshiLabs 2021 Signing Key"
```

Next, we make the binary executable:

```shell
 chmod +x ./Trezor-Suite-XX.XX.X-linux-x86_64.AppImage
 ```

To launch the Trezor suite and connect to our hardware wallet, we run:

```shell
./Trezor-Suite-XX.XX.X-linux-x86_64.AppImage
```

At this point, we can shut down the VM and take a snapshot so we can roll back later if we need to. I like to clone this VM again for each cryptocurrency requiring an external wallet. It increases the security and maintainability of the setup.

## Using the Monero GUI

Besides giving the workstation VM a little more RAM, Monero doesn't require additional setup since Whonix includes the Monero GUI. We launch the app and wait for the Monero daemon to sync, which might take a long time using Tor. After that, we're ready to go!

## Setting Up Adalite

[Adalite](https://adalite.io/) is an open-source, in-browser wallet, and only a few steps are required to get it working.

First, we open the Tor browser and allow pop-up windows from [https://adalite.io](https://adalite.io) by adding an exception for it under **Settings** &rarr; **Preferences** &rarr; **Privacy & Security**.

Next, we launch the Trezor Suite by running `./Trezor-Suite-XX.XX.X-linux-x86_64.AppImage`. It starts the included Trezor Bridge, which Adalite requires to communicate with the Trezor device. We can verify the Trezor Bridge is running by navigating to [http://127.0.0.1:21325/status/](http://127.0.0.1:21325/status/) in the browser.

The final step is to change two advanced settings of the Tor browser by navigating to `about:config` and clicking **Accept the Risk and Continue**. We set the `network.proxy.no_proxies_on` to `127.0.0.1:21325`, so traffic to the Trezor Bridge is not proxied through the Tor network. We also need to disable First-Party Isolation by setting `privacy.firstparty.isolate` to `false`.

We can now use Adalite by navigating to [https://adalite.io](https://adalite.io). It might be a good idea to bookmark that address, so you don't fall victim to a phishing attack in case of a typ-o.

## Closing Thoughts

For my purposes, this setup provides more than enough anonymity. Depending on your needs, it might be overkill or might not be anonymous or secure enough. It always depends on your threat model.

What I don't understand is why SatoshLabs publishes a new signing key every year. Just one more thing to keep in mind come next year.
