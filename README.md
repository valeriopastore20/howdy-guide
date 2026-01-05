# Howdy – Facial authentication for sudo and login
In the latest versions of Fedora, Ubuntu, etc., 'howdy' seems not to work anymore, and the official installation procedure is no longer valid. Here, I'll show you how I installed it and got it working on my Fedora 43. 

I hope this guide can serve as a starting point for updating the original repository.

This guide explains how to install Howdy on Fedora 43 and configure it for authentication with `sudo` and login (GDM)
This guide should also be valid for Ubuntu and derivatives (since it is using Python virtual environments, fixing the dlib installation). Of course, replacing dnf packages with their corresponding apt ones is needed.

---

## 1. Clone Howdy

```bash
git clone https://github.com/boltgolt/howdy.git
cd howdy
```

---

## 2. Install dependencies

```bash
sudo dnf install \
  meson python3 python3-pip python3-setuptools python3-wheel \
  cmake make gcc gcc-c++ redhat-rpm-config \
  pam-devel inih-devel libevdev-devel \
  python3-opencv python3-devel opencv-devel
```

---

## 3. Create a Python virtual environment
> This is needed to install dlib

```bash
python -m venv ~/.local/.howdyvenv
source ~/.local/.howdyvenv/bin/activate
pip install dlib opencv-python
```

---

## 4. Build and install Howdy

```bash
meson setup build
meson compile -C build
sudo meson install -C build
```

---

## 5. Install dlib model data
> These are dlib files needed to perform face recognition

```bash
sudo mkdir -p /usr/local/share/dlib-data
cd /tmp
wget -O shape_predictor_5_face_landmarks.dat.bz2 http://dlib.net/files/shape_predictor_5_face_landmarks.dat.bz2
bunzip2 shape_predictor_5_face_landmarks.dat.bz2
sudo mv shape_predictor_5_face_landmarks.dat /usr/local/share/dlib-data/
wget -O dlib_face_recognition_resnet_model_v1.dat.bz2 http://dlib.net/files/dlib_face_recognition_resnet_model_v1.dat.bz2
bunzip2 dlib_face_recognition_resnet_model_v1.dat.bz2
sudo mv dlib_face_recognition_resnet_model_v1.dat /usr/local/share/dlib-data/
```

---

## 6. Edit Howdy binary

Edit so Howdy always uses the virtual environment instead of the system python packages:

```bash
sudo nano /usr/local/bin/howdy
```

> !!! Adjust the username and Python version if needed.
```sh
#!/bin/sh
/home/USERNAME/.local/.howdyvenv/bin/python \
  "/usr/local/lib64/howdy/cli.py" "$@"
```
CTRL+O to save, CTRL+X to exit.




## 7. Fix Python path for Howdy modules

Edit the compare.py module:

```bash
sudoedit /usr/local/lib64/howdy/compare.py
```

Add at the top:
> !!! Adjust the username and Python version if needed.

```python
import sys
import os

os.environ['PYTHONPATH'] = '/home/USERNAME/.local/.howdyvenv/lib/python3.14/site-packages'
sys.path.insert(0, '/home/USERNAME/.local/.howdyvenv/lib/python3.14/site-packages')
```


---

## 8. Initial Howdy configuration

```bash
sudo howdy config   # select your correct video device
sudo howdy test     # check if working
sudo howdy add      # add a model for face recognition
```

---

## 9. Enable Howdy for sudo

Edit PAM configuration:

```bash
sudo nano /etc/pam.d/sudo
```

Add at the beginning:

```pam
auth    sufficient    /usr/local/lib64/security/pam_howdy.so
```

---

## 10. Enable Howdy for Polkit
```bash
sudo nano /usr/lib/pam.d/polkit-1
```

Add at the beginning:

```pam
auth    sufficient    /usr/local/lib64/security/pam_howdy.so
```

---

## 11. Enable Howdy for GDM (graphical login)
### Here is where things become a little bit trickier. The primary issue is related to the GNOME keyring. As soon as you log in, the keyring is automatically unlocked with the login password. However, now that we are using howdy to log in, the keyring cannot be unlocked automatically with the login password. So, there are diffetent solutions. 
- **Remove the password from the login keyring**  
  Works, but weakens security. Not recommended.

- **Store the keyring password in the TPM and unlock it at login**  
  This is the solution described here. It keeps the keyring protected and relies on the TPM, which is available on most modern systems.  
  Thanks to: https://codeberg.org/umglurf/gnome-keyring-unlock

- **Other methods exist**  
  For example, storing the password in a local “secret” file. This was not tested, as the TPM-based approach was preferred.

In the following, I'm going to show the second one (TPM).
Edit:
```bash
sudo nano /etc/pam.d/gdm-password
```

Example configuration. Note the second line (added) and the commented lines:

```pam
auth     [success=done ignore=ignore default=bad] pam_selinux_permit.so
auth    sufficient	/usr/local/lib64/security/pam_howdy.so

auth        substack	  password-auth
#auth        optional      pam_gnome_keyring.so
auth        include	  postlogin

account     required	  pam_nologin.so
account     include	  password-auth

password    substack	   password-auth
#-password   optional       pam_gnome_keyring.so use_authtok

session     required	  pam_selinux.so close
session     required	  pam_loginuid.so
session     required	  pam_selinux.so open
session     optional	  pam_keyinit.so force revoke
session     required	  pam_namespace.so
session     include	  password-auth
#session     optional      pam_gnome_keyring.so auto_start
session     include	  postlogin
```

Log out and log in. You should see an SELinux denial at this point.

---

## 12. Fix SELinux webcam access (xdm_t → video device)

Check AVC denials:

```bash
sudo ausearch -m AVC,USER_AVC -ts boot --raw
```

Create and install a custom policy:

```bash
sudo ausearch -m AVC -ts recent | audit2allow -M xdm_video
sudo semodule -i xdm_video.pp
```
Log out and log in again.
Gnome should say that the keyring did not unlock. Insert your login password where requested.

---

## 13. Automatic GNOME Keyring unlock with TPM2

### Install dependencies

```bash
sudo dnf install doas
```

### Clone helper

```bash
cd ~/.local
git clone https://codeberg.org/umglurf/gnome-keyring-unlock.git
```

### Configure doas

```bash
sudoedit /etc/doas.conf
```
add:
```conf
permit nopass USERNAME as tss cmd /usr/bin/clevis-encrypt-tpm2
permit nopass USERNAME as tss cmd /usr/bin/clevis-decrypt-tpm2
```

### Encrypt keyring password

```bash
read password #and insert your login password (or the password of the keyring if, in your case, it is different from the login one)
doas -u tss /usr/bin/clevis-encrypt-tpm2 '{"pcr_ids":"7"}' <<<$password > ~/.config/gnome-keyring.tpm2
```

### Test unlock

```bash
doas -u tss /usr/bin/clevis-decrypt-tpm2 < ~/.config/gnome-keyring.tpm2 | ~/.local/gnome-keyring-unlock/unlock.py

```

---

## 14. Automatic keyring unlock at login

Edit `~/.bash_profile`:

```bash
nano ~/.bash_profile
```
add at the end:
```bash
if [ -f ~/.config/gnome-keyring.tpm2 ]
then
    if ! [ -S /run/user/$UID/keyring/control ]
    then
      gnome-keyring-daemon --start --components=secrets
    fi
    doas -u tss /usr/bin/clevis-decrypt-tpm2 < ~/.config/gnome-keyring.tpm2 | ~/.local/gnome-keyring-unlock/unlock.py
fi
```

Logout and login... now the keyring should be unlocked automatically



## 15. References
- https://github.com/boltgolt/howdy
- https://codeberg.org/umglurf/gnome-keyring-unlock.git


## 16. A little thanks
Found this guide useful? Please help me with the bills  
<span class="paypal"><a href="https://www.paypal.me/valeriopastore20" title="Donate to this project using Paypal"><img src="https://www.paypalobjects.com/webstatic/mktg/Logo/pp-logo-100px.png" alt="PayPal donate button" /></a></span>

Happy recognitions.
---


