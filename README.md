![](https://www.yubico.com/wp-content/uploads/2019/10/yubikey_5_family_web_op.png)

# How to set up your Yubikey with macOS Catalina, generate the keys securely and make it work with your SSH client

I'm writing this tutorial because there is little information about how to configure a [Yubikey](https://www.yubico.com/products/yubikey-hardware/) on [macOS Catalina](https://www.apple.com/nl/macos/catalina/), generate the keys securely and make it work with your `ssh` client. 

This tutorial is tested on macOS Catalina version 10.15 with a Yubikey **4** and **5 (NFC)**, [GnuPG 2.2.17](https://gnupg.org/) and [pinetry 1.1.0](https://gnupg.org/related_software/pinentry/index.html). GnuPG2 and Pinetry are installed using the [homebrew](https://brew.sh/) repository. To be able to follow this tutorial I expect the reader to have some knowledge about IT security, Linux and macOS. 

## Requirements
Because we want this to work on our apple machine let's start with installing homebrew. This will give us access to the programs we need. 

```zsh
% /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"`
```

Now that homebrew is installed let's use it to install gnupg2 and pinentry.

```zsh
% brew install gnupg2 pinentry-mac
```

## PIN Codes
The most important thing to start with is configuring your PIN, Admin PIN, and the PUK code.
To do this you must enable admin-mode within `gpg`. See the instructions below on how to enable admin-mode.

```zsh
% gpg --card-edit
gpg/card> admin
Admin commands are allowed
```

Now that we have the correct permissions enter `passwd` to change the PIN, Admin PIN, and the PUK.

```zsh
gpg/card> passwd
```

We are now in the PIN-setup menu. We will start by changing the PIN. 

  * Select option `1` (change PIN).
  * Enter the default PIN code (default: :one::two::three::four::five::six:)
  * Enter your new PIN code.
  * Repeat new PIN code.
  * Check if the PIN actually changes. It should display: `PIN changed`.

Your PIN is now set. Now we need to set the Admin PIN as well.

  * Select option `3` (change Admin PIN).
  * Enter the default Admin PIN code (default: :one::two::three::four::five::six::seven::eight:).
  * Enter your new Admin PIN code.
  * Repeat new Admin PIN code.
  * Check if the PIN actually changes. It should display: `PIN changed`.

Now that we set both the PIN and the Admin PIN we should set the PUK code as well. 

  * Select option `4` (set the Reset Code).
  * Enter the Admin PIN code (which you've just configured).
  * Enter your new PUK code.
  * Repeat new PUK code.
  * Check if the PUK changed. It should display: `Reset Code set`.

You can now choose `Q` to quit the PIN-setup menu.

## Additional configuration on the Yubikey
Additionally, you can configure some other parameters like preferred language, sex, name, and email address for the Yubikey.
At a later stage in this tutorial, we will set the URL parameter.

Just check out the help command in `gpg` and get yourself familiar with the options.

```zsh
gpg/card> help
```

## Generating the keys

### Additional requirements
Before we configure our Mac to work with SSH we'll first focus on generating our keys. Depending on your level of paranoia :scream: you can choose to generate these keys on a separate machine or on the Mac itself.
I use a separate virtual machine, which I can throw away after using it, on my hypervisor at home. After the keys are generated and transmitted to my Yubikey, I'll erase the data and the machine itself.

Some guidelines on how to erase such data can be found in [NIST SP 800-88](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-88r1.pdf). You might think I'm a bit paranoid but as a security professional, I like to familiarize myself with all sorts of security techniques. So while it might not be needed for you or even myself. I like to know how it works. :sunglasses:

If you don't have a hypervisor like me you might want to use something like [QubesOS](https://www.qubes-os.org) to generate the keys. 
Check out [QubesOS on USB](https://www.qubes-os.org/doc/usb-devices/). 

Enough of this. Let's start by generating the keys.

### Generating the primary key
```zsh
% gpg --expert --full-gen-key
```

  * Select option `8` (RSA).
  * Disable encrypt and leave `sign` and `certify` enabled.
  * Select `Q` to continue with the next step.
  * Set the key size to `4096`.
  * Set the expiration date.
  * Setup a UID by entering your `Real Name` and `Email address`. Optionally you can enter some `Comments`.
  * Select `O` for Okay if everything is correct.
  * Setup a passphrase.

After this is done the primary key gets generated. Write down your `<keyID>` because you'll need it in the following steps.
The outcome of the aforementioned steps should look something like this.

```zsh
   pub   rsa4096 <GENERATION DATE> [SC]
         <keyID>
   uid   <YOUR NAME> <XXX@XXX.com>
```

### Other UID's

For the next step, we'll open `gpg` once more only now with the specific goal to edit the key we've just generated.
The <keyID> is displayed a few rules above. 

```zsh
% gpg --expert --edit-key <keyID>`
```

You can now add other UID's by typing in `adduid`. 

```zsh
gpg> adduid
```

Make sure at the end you'll switch back to the primary key by using the following:

```zsh
gpg> uid 1
gpg> primary
```

### Generating the subkeys

#### Encrypt
We will now create the other subkeys and start with the encrypt key.

```zsh
gpg> addkey
```
  * Select option `6` (encrypt only).
  * Set the key size to `4096`.
  * Set the expiration date.
  * Enter your passphrase. 

#### Authenticate 
Next is the authenticate key.

```zsh
gpg> addkey
```
  * Select option `8` (RSA).
  * Disable all and enable `authenticate`.
  * Select `Q` to continue with the next step.
  * Set the key size to `4096`.
  * Set the expiration date.
  * Enter your passphrase. 

:thumbsup: Great all subkeys are created.

### Set the trust level

Now that the second key is generated we should set the trust level to ultimate.

```zsh
gpg> trust
```
  * Select option `5` (I trust ultimately)

```zsh
gpg> save
```

## Add your old signature

If you previously used another key it is advised to sign your new primary key with your previous key. This way you can prove the owner of the new key is the same as that of the old. 

```zsh
% gpg -u <oldKeyID> --sign-key <keyID>
```

## Revocation certificate
Next, we need to create a revocation certificate in case the Yubikey gets lost or the keys get compromised.

```zsh
% gpg --output revoke.asc --gen-revoke <keyID>
```
  * Select option `1` (Key has been compromised)

## Backup the keys
Although you could choose not to create a backup and write these keys to your Yubikey I prefer to back them up myself. Mainly because I want to be able to replace my Yubikey or upgrade them without generating new keys.
Once the private key is stored on the Yubikey it can never be recovered. When you damage your Yubikey this could cause a lot of trouble. That's why I prefer to back them up so I have more control over my keys.
How I back these up securely is something I hold to myself. You can figure it out yourself. :no_mouth:

Having said this we are now going to back up all the keys generated earlier. 

```zsh
% gpg --armor --output my-private-key.sec --export-secret-key <keyID>
% gpg --armor --output my-sub-keys.sec --export-secret-subkeys <keyID>
% gpg --armor --output my-public-key.asc --export <keyID>
```

Save those and protect them!

## Not transferring them to your Mac (but if you must)

You should use the throw-away machine for *writing your keys to Yubikey* but if you somehow can't, you first need to import the newly generated keys into the keyring on your Mac.
This can be done by importing your keys with the `import` option. 

```zsh
% gpg --import my-private-key.sec
% gpg --import my-sub-keys.sec
% gpg --import my-public-key.asc

```

If you didn't write down you <keyID> you can find it by using the `--list-keys` option.

```zsh
% gpg --list-keys
```

## Write your keys to the Yubikey

Now that this is done we can save the keys to the Yubikey by doing the following:

```zsh
% gpg --expert --edit-key <keyID>
gpg> toggle
gpg> keytocard
```
  * Confirm you want to move the primary key.
  * Select option `1` (signature key).
  * Enter your passphrase and the Admin PIN.

```zsh
gpg> key 1
gpg> keytocard
```
  * Select option `2` (encryption key).
  * Enter your passphrase and the PIN.
  * Disable the first key.
```zsh
gpg> key 1
```
  * Enable the second key.

```zsh
gpg> key 2
gpg> keytocard
```
  * Select option `3` (Authentication key).
  * Enter your passphrase and the PIN.
  * Disable the second key and save.
```zsh
gpg> key 2
gpg> save
```

## Stubs

When you write your keys to the Yubikey, locally your keys will be replaced by so-called stubs. These are links to the Yubikey but don't contain any information. 
However, if for some reason something went wrong with the stubs its good to have a backup. After writing the keys to your Yubikey do the following:

```zsh
% gpg --armor --output my-stubs.asc --export-secret-keys <keyID>
```

**Congratulations!** :tada:, your keys are stored on your Yubikey!

## Publish your public key

These days privacy has become very important. Much more if we compare it to 1997 when GnuPG (gpg) was introduced. Back in the days, and even now, there are public keyservers to which you can send your public key. However, there are some privacy concerns with these public keyservers and some nasty ways to attack them :thinking:. **Yakamo K** wrote a good blog post about this which can be found [here](https://medium.com/@mdrahony/are-sks-keyservers-safe-do-we-need-them-7056b495101c).

In recent years an alternative was developed named [keybase.io](https://keybase.io). On this platform, you can publish your public key without publishing your email address and without losing control over your personal information. Because of this, I advise you to use keybase.io. You can even find me on there by clicking [here](https://keybase.io/ronaldvdmeer).

## Define the location of your public key on the Yubikey

Although it's optional you can specify the location of your public key on the Yubicard by doing the following. 

```zsh
> gpg --edit-card
```
  * Type `admin`.
  * Type `url`.
  * Enter the location of your public key.
  * Enter your Admin PIN.

You can now quit `gpg` once more and use the `fetch` command (see 'Fetch the public key') to import them on other machines.

## Using the Yubikey (and your key) on a Windows machine

I often get asked how to use the Yubikey on another machine. Luckily for us, that's not very difficult. 

### Requirements

The steps I'm about to explain are tested on a machine with Microsoft Windows 10 (10.0.17763 Build 17763) installed.
On top of this, you'll also need `gpg` (gpg4win) which can be downloaded by visiting the GnuPG website or by clicking on [this](https://gpg4win.org/download.html) link.
In my experience, you'll also need to install the [Yubico smartcard minidriver](https://www.yubico.com/products/services-software/download/smart-card-drivers-tools/).

After both `gpg4win` and the `mini drivers` are installed you can open the program Kleopatra. This comes with the `gpg4win` installation package. 
This program provides you with a GUI which might give you a better understanding of what is going on.

From the Windows command line (`cmd`) perform the following command:

```zsh
> gpg --edit-card
```

### Fetch the public key

Now, we first need to get ourselves the public key and let `gpg` create the stubs. To get the public key you can manually import it but I prefer to use the `fetch` command.
This command looks at the URL parameter we defined earlier on the Yubikey and downloads your public key. 

```zsh
gpg> fetch
gpg: requesting key from 'https://keybase.io/<YOURKEYBASID>/pgp_keys.asc'
gpg: key XXXXXXXXXX: public key "<YOUR NAME> <XXX@XXX.com>" imported
gpg: Total number processed: 1
gpg:               imported: 1
```

If this is done correctly and you had Kleopatra open at the same time, you should now see the key in your keyring.

### Set the trust level

Now we should set the trust level once more to ultimate. 

```zsh
% gpg --expert --edit-key <keyID>`
gpg> trust
```
  * Select option `5` (I trust ultimately)

```zsh
gpg> save
```

You can use your keys on a Windows machine.

## macOS Configuration for SSH usage

Now that your keys are stored on the Yubikey its time to link `gpg-agent` and `ssh-agent`.
Create the files below and if needed adjust the path to `pinentry`.

Because Apple decided to start using [zsh instead of bash](https://support.apple.com/en-us/HT208050) you'll see me using `.zprofile` instead of `.bash_profile`.
If you're still using `bash` you can make the adjustments to the appropriate files.

```zsh
% cat ~/.gnupg/gpg-agent.conf
pinentry-program /usr/local/bin/pinentry-mac
enable-ssh-support
default-cache-ttl 60
max-cache-ttl 120
```

```zsh
% cat ~/.zprofile
export GPG_TTY="$(tty)"
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
gpgconf --launch gpg-agent
```

After this `kill` all running `gpg` processes, if there are any, and open a new `terminal`. 

```zsh
% kill -9 `ps aux | grep gpg-agent | awk {'print$2'}`
```

Now that this is done you can check to see if the keys are found by `ssh` with the command below:

```zsh
% ssh-add -L
ssh-rsa XXXXXXXXXX cardno:XXXXXXXXXX
```

If this doesn't work check if the `gpg-agent` is running and if not debug why it's not.
However, if it all works you can now start with copying your public key to servers by using `ssh-copy-id`.

```zsh
% ssh-copy-id <username>@<ip>
```

If you encounter trouble check your server to see if the `sshd` configuration accepts public/private key authentication.

## One more thing
The signing of every [commit](https://github.com/RvMp/yubikey-ssh-macos-catalina/commits/master) is done with the Yubikey as well.
If you want to know more about this visit the [Developers Blog](https://developers.yubico.com/PGP/Git_signing.html) on yubico.com. It's fairly simple, yet awesome! :smirk:

# Updates

**2019-11-21** 
Tested this tutorial with the Yubikey 5 (NFC), rephrased some sentences, added an update section and improved markdown.

**2019-11-19** 
Added how to use the Yubikey and GPG keys on another Windows machine.
