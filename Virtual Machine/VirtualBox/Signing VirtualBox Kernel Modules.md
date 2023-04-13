# Signing VirtualBox Kernel Modules

Voici les étapes que j'ai suivies pour activer VirtualBox sur mon ordinateur 
portable **sans désactiver UEFI Secure Boot**. Elles sont presque identiques 
au processus décrit sur le [blog d'Øyvind Stegard][blog], à quelques détails 
clés près. Les images ici sont empruntées au [Wiki UEFI Secure Boot de Systemtap][systemtap].

1. Install the VirtualBox package (this might be different for your platform).
   ```bash
   src='https://download.virtualbox.org/virtualbox/rpm/fedora/virtualbox.repo'
   dst='/etc/yum.repos.d/virtualbox.repo'
   sudo curl ${src} > ${dst}
   dnf check-update
   sudo dnf install VirtualBox-6.0
   ```
2. Create an RSA key pair to sign kernel modules.
   - D'abord on crée les deux variables qui nous servirons dans la commande suivante.
   ```bash
   name="$(getent passwd $(whoami) | awk -F: '{print $1}')"
   out_dir='/root/module-signing'
   ```
   - On crée ensuite le répertoire dans lequel on va stocker nos clès.
   ```bash
   sudo mkdir ${out_dir}
   ```
   - Puis entrons la commande. Nous pouvons le faire en plusieurs paramètres au fur et a mesure, ou en une seule commande.
      - En plusieurs fois :
      ```bash
      sudo openssl \
          req \
          -new \
          -x509 \
          -newkey \
          rsa:2048 \
          -keyout ${out_dir}/MOK.priv \
          -outform DER \
          -out ${out_dir}/MOK.der \
          -days 36500 \  # This is probably waaay too long.
          -subj "/CN=${name}/"
       ```
       - En une seule commande :
       ```bash
       sudo openssl req -new -x509 -newkey rsa:2048 -keyout ${out_dir}/MOK.priv -outform DER -out ${out_dir}/MOK.der -days 36500 -subj "/CN=${name}/"
       ```
    - Puis on change les droits des deux clefs:
   ```bash
   sudo chmod 600 ${out_dir}/MOK*
   ```
   Note the absence of the `-nodes` option from Øyvind's post. With this option
   `openssl` will create a private key with no passphrase. The omission of this
   option prompts for a passphrase, which seems like a good idea for something
   as important as a kernel module signing key.
3. Import the MOK ("Machine Owner Key") so it can be trusted by the system.
   ```bash
   sudo mokutil --import /root/module-signing/MOK.der
   ```
   This will prompt for a password. The password is only temporary and will be
   used on the next boot. It does **not** have to be the same as the signing
   key passphrase.
4. Reboot your machine to enter the MOK manager EFI utility.
   
   - Select _Enroll MOK_.

   ![Enroll MOK][enroll mok]

   - Select _Continue_.

   ![Continue][continue]

   - Select _Yes_ to enroll the keys.

   ![Confirm][confirm]

   - Enter the password from earlier.

   ![Enter password][password]

   - Select _OK_ to reboot.

   ![Reboot][reboot]

   - Verify the key has been loaded by finding the it in the output of
   ```bash
   dmesg | grep '[U]EFI.*cert'
   ```
5. Create a script for signing **all** the VirtualBox kernel modules.
   ```bash
   #!/bin/sh
   
   readonly hash_algo='sha256'
   readonly key='/root/module-signing/MOK.priv'
   readonly x509='/root/module-signing/MOK.der'
   
   readonly name="$(basename $0)"
   readonly esc='\\e'
   readonly reset="${esc}[0m"
   
   green() { local string="${1}"; echo "${esc}[32m${string}${reset}"; }
   blue() { local string="${1}"; echo "${esc}[34m${string}${reset}"; }
   log() { local string="${1}"; echo "[$(blue $name)] ${string}"; }
   
   # The exact location of `sign-file` might vary depending on your platform.
   alias sign-file="/usr/src/kernels/$(uname -r)/scripts/sign-file"
   
   [ -z "${KBUILD_SIGN_PIN}" ] && read -p "Passphrase for ${key}: " KBUILD_SIGN_PIN
   export KBUILD_SIGN_PIN
   
   for module in $(dirname $(modinfo -n vboxdrv))/*.ko; do
     log "Signing $(green ${module})..."
     sign-file "${hash_algo}" "${key}" "${x509}" "${module}"
   done
   ```
   This script differs from Øyvind's in two aspects. First, and most
   importantly, it has :sparkles: **C O L O R S** :sparkles:. Second, it uses
   the magic [`$KBUILD_SIGN_PIN`][kbuild_sign_pin] environment variable that
   doesn't appear _anywhere_ in the `sign-file` usage. I went spelunking in the
   [Linux source][source] for it, but in hindsight I could have just read the
   docs on manual [module signing][module-signing]... I wrote this script to
   `/root/bin/sign-vbox-modules` as that's usually on `root`'s `$PATH`.
6. Execute the aforementioned script as `root`.
   ```bash
   sudo chmod 700 /root/bin/sign-vbox-modules
   sudo -i sign-vbox-modules
   ```
7. Load the `vboxdrv` module.
   ```bash
   sudo modprobe vboxdrv
   ```

[blog]: https://stegard.net/2016/10/virtualbox-secure-boot-ubuntu-fail/
[systemtap]: https://sourceware.org/systemtap/wiki/SecureBoot
[enroll mok]: https://sourceware.org/systemtap/wiki/SecureBoot?action=AttachFile&do=get&target=Screenshot_kvm-rawhide-64-uefi-1_2014-02-27_14_00_13_crop.png
[continue]: https://sourceware.org/systemtap/wiki/SecureBoot?action=AttachFile&do=get&target=Screenshot_kvm-rawhide-64-uefi-1_2014-02-27_14_00_35_crop.png
[confirm]: https://sourceware.org/systemtap/wiki/SecureBoot?action=AttachFile&do=get&target=Screenshot_kvm-rawhide-64-uefi-1_2014-02-27_14_00_44_crop.png
[password]: https://sourceware.org/systemtap/wiki/SecureBoot?action=AttachFile&do=get&target=Screenshot_kvm-rawhide-64-uefi-1_2014-02-27_14_00_53_crop.png
[reboot]: https://sourceware.org/systemtap/wiki/SecureBoot?action=AttachFile&do=get&target=Screenshot_kvm-rawhide-64-uefi-1_2014-02-27_14_01_06_crop.png
[kbuild_sign_pin]: https://github.com/torvalds/linux/blob/12491ed354d23c0ecbe02459bf4be58b8c772bc8/scripts/sign-file.c#L236
[source]: https://github.com/torvalds/linux/blob/12491ed354d23c0ecbe02459bf4be58b8c772bc8/scripts/sign-file.c
[module-signing]: https://www.kernel.org/doc/html/v4.20/admin-guide/module-signing.html#manually-signing-modules
