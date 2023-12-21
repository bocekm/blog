---
layout: post
title:  "Using Nitrokey 3 for sudo, Gnome login, Kerberos and VPN on Fedora"
date:   2023-11-8 22:43:37 +0200
---
A [Nitrokey](https://www.nitrokey.com/products/nitrokeys) is an open-source hardware and firmware alternative to closed-source Yubikeys. I've tested the steps with Fedora 38 and a [Nitrokey 3C NFC harware token](https://shop.nitrokey.com/shop/nk3cn-nitrokey-3c-nfc-148).

## Add udev rules for the Nitrokey HW

Source: [https://docs.nitrokey.com/software/nitropy/linux/udev]()

```
$ wget https://raw.githubusercontent.com/Nitrokey/libnitrokey/master/data/41-nitrokey.rules
$ sudo mv 41-nitrokey.rules /etc/udev/rules.d/
$ sudo udevadm control --reload-rules && sudo udevadm trigger
```

## Install a cli tool for the Nitrokey

Source: [https://docs.nitrokey.com/software/nitropy/all-platforms/installation#preparation]()

```
$ python3 -m ensurepip
$ python3 -m pip install --user pipx
$ python3 -m pipx ensurepath
$ pipx install pynitrokey
```

## Upgrade the Nitrokey firmware

Source: [https://docs.nitrokey.com/nitrokey3/linux/firmware-update]()

```
$ pipx upgrade pynitrokey
$ nitropy nk3 update
```

## Allow the Nitrokey for sudo and logging in to Gnome

Source: [https://fedoramagazine.org/use-fido-u2f-security-keys-with-fedora-linux/]() and [https://docs.nitrokey.com/nitrokey3/linux/desktop-login]()

```
$ sudo dnf install pam-u2f pamu2fcfg authselect
$ authselect current
$ sudo authselect select sssd with-pam-u2f
$ mkdir ~/.config/Nitrokey
$ pamu2fcfg > ~/.config/Nitrokey/u2f_keys
$ sudo mv ~/.config/Nitrokey /etc
```

Add the following line to _/etc/pam.d/sudo_ and _/etc/pam.d/gdm-password_ to enable the Nitrokey authentication for sudo and Gnome login:

```
auth sufficient pam_u2f.so authfile=/etc/Nitrokey/u2f_keys cue prompt nouserok
```

## Set up a Nitrokey PIN

Source: [https://docs.nitrokey.com/nitrokey3/windows/set-pins#passwords-and-otp-secrets]()

Setting a PIN is optional but highly recommended. If you don't set up a PIN to be entered every time you press Nitrokey and you leave your device and Nitrokey unattended, then your system, and potentially other systems you connect to through VPN, could be compromised.

```
$ nitropy nk3 secrets set-pin
```

## Set up HOTP in the Nitrokey

Source: [https://github.com/Nitrokey/pynitrokey/pull/393]()

```
$ wget https://git.sr.ht/~mcepl/gen-oath-safe/blob/master/gen-oath-safe
$ sudo install -Dm755 ./gen-oath-safe /usr/local/bin/gen-oath-safe 
$ gen-oath-safe nitro hotp
INFO: No secret provided, generating random secret.

Key in Hex: 4e414f554258434d4e595452515754465a554a44
Key in b32: JZAU6VKCLBBU2TSZKRJFCV2UIZNFKSSE (checksum: 3)
```

Get the secret for nitropy from the line `Key in b32: JZAU6VKCLBBU2TSZKRJFCV2UIZNFKSSE`.

Set up a new HOTP number generator in the Nitrokey:

```
$ nitropy nk3 secrets add-otp --kind HOTP --digits-str 6 --hash SHA1 --touch-button mycompanyhotp --protect-with-pin JZAU6VKCLBBU2TSZKRJFCV2UIZNFKSSE
$ nitropy nk3 secrets get mycompanyhotp
```

Note: If you don't use a PIN for getting Nitrokey secrets, do not use the --protect-with-pin option in the register command above.

Delete your shell history after performing the above register step to not leave the secret in the history in a clear text.

```
$ history  # print all the shell history entries and find the command that contains the secret
$ history -d <entry number>  # delete the sensitive entry
```

## Connect to VPN

The following setup is for cases where connecting to a VPN requires a combination of a password and an HOTP code.
```
$ read -sp 'Password: ' pswd; nitropy nk3 secrets get mycompanyhotp | awk -v pswd=$pswd 'NR==3 {print pwsd$0; exit}' | nmcli con up "My preconfigured VPN profile" --ask
```

Note: If you don't use a PIN for getting Nitrokey secrets, use _NR==2_ instead of _NR==3_ in the command above.

## Store your kerberos password in Nitrokey

```
$ nitropy nk3 secrets add-password --touch-button --protect-with-pin --password <your kerberos password> kerberos
```

Note: If you don't use a PIN for getting Nitrokey secrets, leave the _--protect-with-pin_ option out of the command above.

## Get a new kerberos ticket

```
$ nitropy nk3 secrets get-password kerberos | awk 'NR==3 {print $3; exit}' | kinit <username>@<domain>
```

Note: If you don't use a PIN for getting Nitrokey secrets, use _NR==2_ instead of _NR==3_ in the command above.

## Create command line aliases

For Bash shell, add the bellow lines to ~/.bashrc.

```
alias vpn="read -sp 'Password: ' pswd; nitropy nk3 secrets get mycompanyhotp | awk -v pswd=\$pswd 'NR==3 {print pwsd\$0; exit}' | nmcli con up \"My preconfigured VPN profile\" --ask"
alias kn="nitropy nk3 secrets get-password kerberos | awk 'NR==3 {print \$3; exit}' | kinit <username>@<domain>"
```

Note: If you don't use a PIN for getting Nitrokey secrets, use _NR==2_ instead of _NR==3_ in both commands above.