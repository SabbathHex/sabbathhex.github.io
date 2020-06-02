---
layout: post
title: Firefox Browserpass with firejail and AppArmor
---
This is a guide describing how to set up Firefox with [firejail](https://github.com/netblue30/firejail/) + [AppArmor](https://en.wikipedia.org/wiki/AppArmor) with password management in [pass](https://www.passwordstore.org/). The passwords are filled in websites forms through [Browserpass addon](https://addons.mozilla.org/en-US/firefox/addon/Browserpass-ce/) + [Browserpass](https://github.com/Browserpass/Browserpass-native) native application.

The setup password store will only contain passwords used by the websites encrypted through a separate key.

<!--more-->
* Do not remove this line (it will not be displayed)
{:toc}

# Setup

## Creating new .gnupg directory

First, the separate `.gnupg` directory needs to be set up. This can be done by following the ["Creating a key"](https://www.dewinter.com/gnupg_howto/english/GPGMiniHowto-3.html#ss3.1) tutorial, but the `gpg` command should be run with `--homedir=~/.gnupg_ff` option.

    (user) ~ # gpg --homedir=~/.gnupg_ff --gen-key

## Create the new password store

The browser-only password store may be set up at `${HOME}/.websites-passwords`. Pass can set up several stores with `PASSWORD_STORE_GPG_OPTS` and `PASSWORD_STORE_DIR` variables, e.g.:

    (user) ~ # PASSWORD_STORE_GPG_OPTS="--homedir=${HOME}/.gnupg_ff" PASSWORD_STORE_DIR=".websites-passwords" pass init

## Configuring the Browserpass native application

At the time of writing, Browserpass addon does not support running `gpg` with predefined parameters, so it is possible to either:

* Create a wrapper and place it somewhere where it would be deemed executable by `AppArmor`. For example, as `/usr/bin/gpg_forced_homedir`, with the following content:

```shell
#!/bin/sh
# Since firejail will probably strip the environment variables, replace with real path to ${HOME}, like /home/larry
/usr/bin/gpg2 --homedir="${HOME}/.gnupg_ff" $@
```
* Use a small patch to Browserpass native application (replace `/home/larry` with a real path):

```
diff --git a/request/fetch.go b/request/fetch.go
index fd35610..83d4259 100644
--- a/request/fetch.go
+++ b/request/fetch.go
@@ -154,7 +154,7 @@ func decryptFile(store *store, file string, gpgPath string) (string, error) {
    }

    var stdout, stderr bytes.Buffer
-	gpgOptions := []string{"--decrypt", "--yes", "--quiet", "--batch", "-"}
+	gpgOptions := []string{"--homedir=/home/larry/.gnupg_ff", "--decrypt", "--yes", "--quiet", "--batch", "-"}

    cmd := exec.Command(gpgPath, gpgOptions...)
    cmd.Stdin = passwordFile
```

If using Gentoo, make sure to use the built-in patch management, see [/etc/portage/patches](https://wiki.gentoo.org/wiki//etc/portage/patches).

## Configuring addon
If you installed the `gpg_forced_homedir` wrapper, configure the Browserpass addon to use the wrapper as the custom gpg binary in addon Preferences.

In any case, the new store will need to be specified in the Customer store location

## Modifying the firejail profile

Create a local copy of the profile as per the [firejail instructions](https://firejail.wordpress.com/documentation-2/building-custom-profiles/). Include the following in the copy:


```shell
mkdir ${HOME}/.gnupg_ff
whitelist ${HOME}/.gnupg_ff
mkdir ${HOME}/.websites-passwords
whitelist ${HOME}/.websites-passwords
```


### Note on D-Bus
If the only available pinentry programs are gnome ones, I believe `ignore nodbus` is required in the profile, otherwise the pinentry window would not launch. As an alternative, `pinentry-fltk` may be used(available on Gentoo with `USE="ftlk"` for `app-crypt/pinentry`). D-Bus is by default disabled in the Firefox firejail profile.

## Configuring gpg-agent
Set the needed pinentry in `${HOME}/.gnupg_ff/gpg-agent.conf`:

```
pinentry-program /usr/bin/pinentry-fltk
```

## Modifying the AppArmor profile

AppArmor should allow executables to be run from `/usr/libexec/Browserpass-native`

```
(user) ~ # cat /etc/AppArmor.d/local/firejail-local
# Site-specific additions and overrides for 'firejail-default'.
# For more details, please see /etc/AppArmor.d/local/README.
/{,run/firejail/mnt/oroot/}{,usr/,usr/local/}libexec/Browserpass-native ix,
```

# Integrating with the standard pass directory

As an optional measure, it is possible to configure the usual `pass` to the passwords created in the second store. This can be automated if the standard `.password-store` is set up as a git repo (`pass git init`) and is easier if all Browserpass passwords are in a separate directory, e.g. `.password-store/site-passwords`.

First, import the key from `.gnupg_ff` into the keyring at `.gnupg`.

Back up the password storage directory (if not using a remote repository) and run `gpg --list-keys` to determine the newly imported key fingerprint.

To propagate changes done in `.password-store/site-passwords` to `.websites-passwords` password store, create a [git post-commit hook](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) that would rsync the changed `site-passwords` subdirectory into `.websites-passwords`.

```
(user) ~/.password-store # echo > .git/hooks/post-commit <<EOF
#!/bin/sh
rsync -avh site-passwords/ ${HOME}/.websites-passwords --delete
EOF
```

The `-avh` with `--delete` will make `${HOME}/.websites-passwords/` an exact copy of `site-passwords/` subdirectory.

The following command will reencrypt the specified directory with the new key. Run `pass init -p site-passwords <fingerprint>` where `site-passwords` is the directory with the passwords to be used with Browserpass.

For more info on how to backup `.password-store` as a git repo, see [this gist](https://gist.github.com/abtrout/d64fb11ad6f9f49fa325) by abtrout. The post-commit hook may be expanded to automatically run `git push`

If you want to manage the new password store separately, you can add an alias to your shell:

    alias wpass="PASSWORD_STORE_GPG_OPTS='--homedir=~/.gnupg_ff' PASSWORD_STORE_DIR='.websites-passwords' pass"

# Further configuration and notes
The sandbox may be separated from the host even further by running it in Xephyr as in [this guide by Sakaki](https://wiki.gentoo.org/wiki/Sakaki's_EFI_Install_Guide/Sandboxing_the_Firefox_Browser_with_Firejail#user_namespaces), however there are a few minor problems:

* Clipboard synchronization relies on `xsel`, which does not support images in the clipboard, so a screenshot in the clipboard will not be synchronized.

    A possible solution is to replace `xsel` with `xclip`, however that may require moving to a different language, since `bash` seems to have trouble processing the clipboard data, complaining about  `warning: command substitution: ignored null byte in input`.
* Xephyr by default has `Ctrl`+`Shift` configured to capture input inside the window. Using Firefox shortcuts that involve those keys has the unwanted side effect. This can be adjusted by adding `-no-host-grab` to `xephyr-extra-params` in `/etc/firejail/firejail.config`
* There is no keyboard layout synchronization between guest and host by default
