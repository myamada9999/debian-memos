# TP-Link Archer T3U Nano on Debian with Secure Boot

Verified: Debian kernel `6.1.0-53-amd64`, USB ID `2357:012e`, DKMS `rtl88x2bu/5.13.1`; Wi-Fi connectivity confirmed.

## Driver setup

```bash
lsusb
sudo apt update
sudo apt install -y git dkms build-essential linux-headers-$(uname -r)
git clone https://github.com/morrownr/88x2bu-20210702.git
cd 88x2bu-20210702
sudo ./install-driver.sh
dkms status
```

The USB device appeared in `lsusb` but initially did not appear in `ip link`. `sudo modprobe 88x2bu` initially failed with `Key was rejected by service`, since Secure Boot was enabled.

## MOK enrollment and signing

The original `/var/lib/dkms/mok.key` and `mok.pub` files were **zero bytes**. Only if they are empty, back them up:

```bash
sudo mv /var/lib/dkms/mok.key /var/lib/dkms/mok.key.empty
sudo mv /var/lib/dkms/mok.pub /var/lib/dkms/mok.pub.empty
```

Generate a private key and DER X.509 certificate:

```bash
sudo openssl req -new -x509 -newkey rsa:3072 \
  -keyout /var/lib/dkms/mok.key \
  -out /var/lib/dkms/mok.pub \
  -nodes -days 3650 \
  -subj "/CN=Debian DKMS Module Signing/" -outform DER
sudo chmod 600 /var/lib/dkms/mok.key
```

Note: `dkms generate_mok` was not supported by the installed version.

Sign the installed kernel module:

```bash
sudo /usr/src/linux-headers-$(uname -r)/scripts/sign-file \
  sha256 /var/lib/dkms/mok.key /var/lib/dkms/mok.pub \
  /lib/modules/$(uname -r)/updates/dkms/88x2bu.ko
modinfo -F signer 88x2bu
```

Expected signer: `Debian DKMS Module Signing`.

```bash
sudo mokutil --import /var/lib/dkms/mok.pub
sudo reboot
```

At the MOK Manager screen, choose Enroll MOK → Continue → Yes → enter temporary password → Reboot.

```bash
sudo mokutil --test-key /var/lib/dkms/mok.pub
sudo modprobe 88x2bu
nmcli device status
```

The new interface was `wlx105a95b13a06` and connecting to Wi-Fi succeeded.

## Automatically signing after DKMS rebuilds

DKMS can use the same enrolled MOK for future module builds, but the exact configuration depends on the installed DKMS version. Inspect settings before editing:

```bash
dkms --version
grep -R -n -E '^(mok_signing_key|mok_certificate|sign_file|sign_tool)=' /etc/dkms /etc/dkms.conf 2>/dev/null
```

For DKMS versions supporting `mok_signing_key` and `mok_certificate`, configure **existing** paths in `/etc/dkms/framework.conf` (do not duplicate conflicting entries):

```bash
mok_signing_key="/var/lib/dkms/mok.key"
mok_certificate="/var/lib/dkms/mok.pub"
```

Verify after the *next* kernel update/rebuild:

```bash
dkms status
modinfo -k "$(uname -r)" -F signer 88x2bu
sudo modprobe 88x2bu
nmcli device status
```

The two settings are guidance to verify against the installed DKMS version; automatic signing was **not tested** in this session. Do not commit `mok.key`, Wi-Fi credentials, or machine-specific secrets.
