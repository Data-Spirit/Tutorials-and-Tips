#

Voici les étapes que j'ai suivies pour activer VirtualBox sur mon ordinateur 
portable **sans désactiver UEFI Secure Boot**. Elles sont presque identiques 
au processus décrit sur le [blog d'Øyvind Stegard][blog], à quelques détails 
clés près. Les images ici sont empruntées au [Wiki UEFI Secure Boot de Systemtap][systemtap].

---

### 1 - Installation de VirtualBox

1. Installation du paquet `VirtualBox` (cela peut différer selon votre plateforme).<br>
   (Vous pouvvez avoir plus de détails en consultant la documentation [Ubuntu-fr.org/VirtualBox][Ubuntu_install_vbox]).
   
   <details>
   <summary>Installation par les dépôts officiels d'Ubuntu</summary>
   Pour installer VirtualBox tel qu'empaqueté par l'équipe d'Ubuntu, installez les paquets :<br>
   <code>virtualbox</code> <code>virtualbox-qt</code> <code>virtualbox-dkms</code> <code>virtualbox-guest-additions-iso</code> <code>virtualbox-guest-utils</code>.<br>
   <br>
   <pre lang="bash">sudo apt-get install virtualbox virtualbox-qt virtualbox-dkms virtualbox-guest-additions-iso virtualbox-guest-utils</pre>
   </details>

   <details>
   <summary>Installation depuis le dépôt d'Oracle (version la plus à jour)</summary>
   Pour installer l'édition de VirtualBox telle que proposée par Oracle, vous devez ajouter son dépôt à votre liste de sources de logiciels ainsi que sa clé de signature.<br>
   Puis, vous procédez à l'installation de virtualbox-6.1.<br>
   <br>
   
   - Dans une fenêtre de terminal, exécutez la commande suivante afin de récupérer les clés de signature du dépôt de VirtualBox :<br>
   <pre lang="bash">wget -q -O- http://download.virtualbox.org/virtualbox/debian/oracle_vbox_2016.asc | sudo apt-key add -</pre><br>
   
   - Ajoutez le dépôt d'Oracle compatible avec votre version d'Ubuntu à votre liste de sources de logiciels en exécutant la commande suivante dans un terminal :<br>
   <pre lang="bash">echo "deb [arch=amd64] http://download.virtualbox.org/virtualbox/debian $(lsb_release -sc) contrib" | sudo tee /etc/apt/sources.list.d/virtualbox.list</pre><br>
   
   - Rechargez la liste des paquets disponibles pour installation en exécutant la commande suivante dans un terminal :<br>
   <pre lang="bash">sudo apt-get update</pre><br>
   
   - Pour connaître la dernière version installable :<br>
   <pre lang="bash">apt-cache madison virtualbox</pre><br>
   
   - Installation du paquet virtualbox-*. (remplacer "*" par la dernière version trouvée avec la commande précédente).<br>
   <pre lang="bash">sudo apt-get install virtualbox-6.1</pre><br>
   </details>

   <details>
   <summary>Ajouts de d'autres méthodes</summary>
   Ajout futur eventuel pour d'autres méthodes.<br>
   <pre lang="bash">Futur commandes</pre><br>
   </details>

---

### 2 - Signature des modules du noyau de VirtualBox.
1. Création des clès de chiffrement : Création d'une paire de clés RSA pour signer les modules du noyau.

   - D'abord on crée les deux variables qui nous servirons dans la commande suivante.
   ```bash
   name="$(getent passwd $(whoami) | awk -F: '{print $1}')"
   ```
   ```bash
   out_dir='/root/module-signing'
   ```

   - On crée ensuite le répertoire dans lequel on va stocker nos clès.
   ```bash
   sudo mkdir ${out_dir}
   ```

   - Puis entrons la commande. Nous pouvons le faire en plusieurs paramètres au fur et a mesure, **OU** en une seule commande.
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

      ⮕ Version de la *commande unique* **SANS** le paramètre `-nodes`:
       ```bash
       sudo openssl req -new -x509 -newkey rsa:2048 -keyout ${out_dir}/MOK.priv -outform DER -out ${out_dir}/MOK.der -days 36500 -subj "/CN=${name}/"
       ```
       
      ⮕ Version de la *commande unique* **AVEC** le paramètre `-nodes`:
       ```bash
       sudo openssl req -new -x509 -newkey rsa:2048 -keyout ${out_dir}/MOK.priv -outform DER -out ${out_dir}/MOK.der -nodes -days 36500 -subj "/CN=${name}/"
       ```
    - Puis on change les droits des deux clefs:
   ```bash
   sudo chmod 600 ${out_dir}/MOK*
   ```
   
   ⚠️ Attention ⚠️ :<br>
   **AVEC** l'option `-nodes`, `openssl` va créer une clé privée **SANS** passphrase.<br>
   **SANS** l'option `-nodes`, `openssl` va créer une clé privée **AVEC** passphrase.   

   **Le retrait de cette option invitant donc à entrer une passphrase, semble être une 
   bonne idée pour quelque chose d'aussi important qu'une clé de signature de module 
   de noyau.**


2. Importer la clé MOK ("Machine Owner Key") afin qu'elle puisse être considérée 
   comme fiable par le système.

   ```bash
   sudo mokutil --import /root/module-signing/MOK.der
   ```
   Cela vous demandera un mot de passe. Le mot de passe est temporaire et 
   sera utilisé lors du prochain démarrage. Il n'est **pas** obligatoire 
   que ce soit le même que celui de la phrase secrète de la clé de signature.


3. Redémarrez votre machine pour accéder à l'utilitaire de gestion MOK EFI.

   - Selectionner _Enroll MOK_.

   ![Enroll MOK][screen-enroll mok]

   - Selectionner _Continue_.

   ![Continue][screen-continue]

   - Selectionner _Yes_ to enroll the keys.

   ![Confirm][screen-confirm]

   - Enter the mot de passe défini plus tôt **(ne pas confondre avec la phrase secrète)**.

   ![Enter password][screen-password]

   - Selectionner _OK_ to reboot.

   ![Reboot][screen-reboot]

   - Vérifiez que la clé a été chargée en la cherchant dans la sortie de

   ```bash
   dmesg | grep '[U]EFI.*cert'
   ```

---

### 3 - Création d'un script pour l'automatisation de la procédure.

1. Création d'un script pour signer **tous** les modules du noyau de VirtualBox.
   - création du fichier pour y incorporer le script (enlever sudo si vous êtes déjà en `root`).<br>
   (Vous pouvez aussi remplacer nano par le nom du programme de votre editeur de texte).
   ```bash
   sudo nano /usr/bin/sign-vbox-modules
   ```

   - Puis ajoutez dans le fichier `sign-vbox-modules` le script suivant :
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

   Ce script diffère de celui d'Øyvind sur deux aspects. Premièrement, et surtout, 
   il dispose de :sparkles: **C O U L E U R S** :sparkles:. Deuxièmement, il 
   utilise la variable d'environnement magique [`$KBUILD_SIGN_PIN`][kbuild_sign_pin] 
   qui n'apparaît _nulle part_ dans l'utilisation de `sign-file`. J'ai fouillé dans 
   le [Linux source][source] de Linux pour la trouver, mais avec du recul j'aurais 
   pu simplement lire la documentation sur la [signature des modules][module-signing]... 
   J'ai écrit ce script dans le fichier `/usr/bin/sign-vbox-modules` car c'est 
   généralement sur le `$PATH` de `root`.

   
2. Exécution du script susmentionné en tant que `root`.

   - Rendre le fichier executable par `root`.
   ```bash
   sudo chmod 700 /root/bin/sign-vbox-modules
   ```

   -Puis l'executer en tant que `root`.
   ```bash
   sudo -i sign-vbox-modules
   ```

3. Load the `vboxdrv` module.
   ```bash
   sudo modprobe vboxdrv
   ```

---

### 4 - Liens utiles.





---

[blog]: https://stegard.net/2016/10/virtualbox-secure-boot-ubuntu-fail/
[systemtap]: https://sourceware.org/systemtap/wiki/SecureBoot
[screen-enroll mok]: https://sourceware.org/systemtap/wiki/SecureBoot?action=AttachFile&do=get&target=Screenshot_kvm-rawhide-64-uefi-1_2014-02-27_14_00_13_crop.png
[screen-continue]: https://sourceware.org/systemtap/wiki/SecureBoot?action=AttachFile&do=get&target=Screenshot_kvm-rawhide-64-uefi-1_2014-02-27_14_00_35_crop.png
[screen-confirm]: https://sourceware.org/systemtap/wiki/SecureBoot?action=AttachFile&do=get&target=Screenshot_kvm-rawhide-64-uefi-1_2014-02-27_14_00_44_crop.png
[screen-password]: https://sourceware.org/systemtap/wiki/SecureBoot?action=AttachFile&do=get&target=Screenshot_kvm-rawhide-64-uefi-1_2014-02-27_14_00_53_crop.png
[screen-reboot]: https://sourceware.org/systemtap/wiki/SecureBoot?action=AttachFile&do=get&target=Screenshot_kvm-rawhide-64-uefi-1_2014-02-27_14_01_06_crop.png
[kbuild_sign_pin]: https://github.com/torvalds/linux/blob/12491ed354d23c0ecbe02459bf4be58b8c772bc8/scripts/sign-file.c#L236
[source]: https://github.com/torvalds/linux/blob/12491ed354d23c0ecbe02459bf4be58b8c772bc8/scripts/sign-file.c
[module-signing]: https://www.kernel.org/doc/html/v4.20/admin-guide/module-signing.html#manually-signing-modules

[Ubuntu_install_vbox]: https://doc.ubuntu-fr.org/virtualbox#installation
