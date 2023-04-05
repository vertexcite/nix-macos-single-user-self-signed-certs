# Workaround for Nix/macOS self-signed HTTPS certs problem

Date: 2023-04-06

## The problem
Some organisations setup firewalls to intercept HTTPS, and use self-signed certificates.  macOS nix users seem to be struggling when their connection to the internet is via when this is the case. It doesn't seem possible to get the nix-daemon to trust self-signed certs.  The environment variable to make nix trust self-signed cert doesn't work on macOS, because the environment variable doesn't get through to the nix daemon, and only multi-user installs are supported on macOS.  

## Workaround
The `NIX_SSL_CERT_FILE` environment variable works for single user install, allowing nix to trust self-signed certs. It should be possible to get a single user install working to some degree. The following is the story of what at the time of writing seems to have been a successful attempt. It is a mix of first doing a multi user install, then removing some of what that install did and doing a single user install.  It is admittedly a messy workaround, since it leaves a lot of unecessary pieces lying around from the multi user install.

## Overview
Pre-requisites:
* You need to be able to use sudo (i.e. have admin rights)
* You need a `.pem` file that contains the self-signed certificate that you want nix to trust.  In the notes below, that is assumed to be saved at `/Users/<your_user>/self-signed-certificates-ssl-https.pem`
	
Steps:
1) Do normal macOS multi-user install of nix
1) Clear out contents of `/nix`
1) Change `/nix` to be owned by your user
1) Modify macOS specific nix install to do a single user install
1) Set environment variables in shell startup script
1) Set Nix channels

## Steps in more detail
Do multi user install as per https://nixos.org/download.html#nix-install-macos

On macOS, that runs this script: https://releases.nixos.org/nix/nix-2.13.3/install, which in turn downloads a macOS specific script from https://releases.nixos.org/nix/nix-2.13.3/nix-2.13.3-aarch64-darwin.tar.xz, we'll come back to this in a moment.

Once that's done, clear out content within `/nix`.  Make sure you get this right, otherwise this could make a big mess of things.
```
cd /nix
sudo rm -rf * # BE CAREFUL!
```

Change the ownership of `/nix` to your user
```
sudo chown -R <your_user> /nix
```
E.g., if your username is "alice" then it would be:
```
sudo chown -R alice /nix
```

Download that macOS specific script, mentioned earlier (i.e. https://releases.nixos.org/nix/nix-2.13.3/nix-2.13.3-aarch64-darwin.tar.xz ), 
and decompress (`xz -d nix-2.13.3-aarch64-darwin.tar.xz`), (get `xz` using homebrew, or use nix in docker and `nix-shell -p xz`)

Then untar (e.g., `tar -xf nix-2.13.3-aarch64-darwin.tar`)

Then edit the macOS specific install script to force single user install, e.g.:
```
cd nix-2.13.3-aarch64-darwin
cp install install-single-user
vi install-single-user
```
Make edits on lines 46 and 64 so that diff looks like this:
```
diff install install-single-user
46c46
<         INSTALL_MODE=daemon;;
---
>         INSTALL_MODE=no-daemon;;
64c64
<                 exit 1
---
>                 # exit 1
```

Run install-single-user (i.e. `./install-single-user`)

Add these lines to `~/.zshrc` (probably similar needed if you use bash)
```
export NIX_PATH=/Users/<your_user>/.nix-defexpr/channels
export NIX_SSL_CERT_FILE=/Users/<your_user>/self-signed-certificates-ssl-https.pem
```

Then, open a new terminal window.

Then, setup channels:

See https://stackoverflow.com/questions/73164546/error-file-nixpkgs-was-not-found-in-the-nix-search-path-add-it-using-nix-pa

And do this:
```
nix-channel --add https://nixos.org/channels/nixpkgs-unstable
nix-channel --update
```

Now, things like this should work:
```
nix-shell -p lynx
```


Good luck.
