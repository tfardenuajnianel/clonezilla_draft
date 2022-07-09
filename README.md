# clonezilla_draft

[Version publiée](https://github.com/lenainjaune/clonezilla/edit/main/README.md)

# En cours
```sh
# [1] utiliser les services (https://unix.stackexchange.com/questions/529111/execute-a-script-before-login-screen/529183#529183)

# TODO : script internes par function ne marche pas comme attendu
# TODO : bug export NBD depuis systeme windows
# TODO : echo -e apparemment pose pb 
# TODO : démarrer SSH ailleurs que /etc/profile exécuté à chaque logon
# TODO : si script utilisateur absent en recreer un par defaut ; si existe deja et que c'est un bien un fichier texte verifier que shebang en ligne 1 
# TODO : parfois on doit redémarrer avahi (hostname-2 !) => https://github.com/lathiat/avahi/issues/117 (tester en particulier cache-entries-max=0 de gramels qui contournerait mais empêcherait la résolution à son tour)
# TODO : idee DBG* -> setVar MDBG_NO_DEBUG	1
# TODO : si DBG inner system only on chroot et on quitte
# TODO : DBG pour empecher apt update


# Définir le support (pour une clé USB `dmesg -w` avant de brancher)
lsblk
DEV=/dev/sdb


# Configurer meta debug

MDBG_NO_DEBUG=0
MDBG_MODIFY_INNER_SYSTEM=1


# Configurer debug (uniquement si MDBG_* != 1)

DBG_NO_INSTALL_DEP_HOST=1
DBG_NO_FORMAT=1
DBG_NO_DOWNLOAD_ISO=1
DBG_NO_EXTRACT_ISO=1
DBG_NO_MAKE_BOOTABLE=1


# Configurer meta debug

if [[ $MDBG_MODIFY_INNER_SYSTEM -eq 1 ]] ; then

  DBG_NO_INSTALL_DEP_HOST=1
  DBG_NO_FORMAT=1
  DBG_NO_DOWNLOAD_ISO=1
  DBG_NO_EXTRACT_ISO=1
  DBG_NO_MAKE_BOOTABLE=1

fi
# NO_DEBUG prioritaire (il DOIT être le dernier)
if [[ $MDBG_NO_DEBUG -eq 1 ]] ; then

  MDBG_MODIFY_INNER_SYSTEM=0 

  DBG_NO_INSTALL_DEP_HOST=0 
  DBG_NO_FORMAT=0 
  DBG_NO_DOWNLOAD_ISO=0 
  DBG_NO_EXTRACT_ISO=0 
  DBG_NO_MAKE_BOOTABLE=0 

fi


# Installer dépendances hôte (Debian) en SU

if [[ $DBG_NO_INSTALL_DEP_HOST -ne 1 ]] ; then

 apt update && apt install -y \
  rsync libc6-i386 mtools squashfs-tools parted wget dosfstools

fi


# Configurer support

DNS=8.8.8.8
HOSTNAME_SUPPORT=CZ-LIVE
PART_RW_NAME=casper-rw
## 'PASS' protège PASS d'une expansion non voulue de * ou ! etc.
PASS='P455w0rd!*'
INNER_1ST_SCRIPT=inner_1st_script.sh
INNER_PASSWORD_MODIFIER=password_modifier.sh
INNER_HOSTNAME_MODIFIER=hostname_modifier.sh
OUTER_1ST_SCRIPT=ran_first.sh
SCRIPT_NAME_NBD_EXPORT=export_all_local_disk.sh
#ISO_ARCH="(i386|i486|i686)"
ISO_ARCH=64
#USB_PART1_SIZE=1G
USB_PART1_SIZE=90%
USB_PART2_SIZE=100%


# Préparer support

if [[ $DBG_NO_FORMAT -ne 1 ]] ; then

 parted -a cylinder $DEV -s "mklabel msdos" \
  -s "mkpart primary fat32 1 $USB_PART1_SIZE" \
  -s "mkpart primary ext4 $USB_PART1_SIZE $USB_PART2_SIZE"
 mkfs.vfat -n $HOSTNAME_SUPPORT -I -F 32 ${DEV}1
 mkfs.ext4 -F -L $PART_RW_NAME ${DEV}2

fi

# Nota : il manque qqc ; lsblk n'actualise pas FSTYPE,FSVER,LABEL de DEV
#  => on le voit si formatage précédent par "dd ISO /dev/$DEV" 


# Télécharger dernier ISO stable réel (nécessite mtools pour CZ récents)

if [[ $DBG_NO_DOWNLOAD_ISO -ne 1 ]] ; then

  ISO_DL_SOURCE=\
https://sourceforge.net/projects/clonezilla/files/clonezilla_live_stable

  cz_latest_dl_folder=$(
   wget --no-check-certificate -qO- $ISO_DL_SOURCE \
   | grep -Eo "https?://[^ \"]*sourceforge[^ \"]*/[-0-9.]+/" \
   | sort -rV | head -n1
  )
  cz_latest_dl_version=$(
   echo $cz_latest_dl_folder | grep -Eo "[-0-9.]{2,}" 
  )
  cz_latest_iso=$(
   wget --no-check-certificate -qO- $cz_latest_dl_folder \
   | grep -iEo "https?://[^ \"]*sourceforge[^ \"]*\.iso" \
   | grep -E $cz_latest_dl_version | grep -E "$ISO_ARCH[^-]" | sort -u
  )
  wget --no-check-certificate $cz_latest_iso -O cz_latest.iso

else

 ln $( ls -1 clonezilla-live* | sort -V | tail -n 1 ) cz_latest.iso

fi

# Monter support

mkdir -p /mnt/CLONEZILLA/ /mnt/$PART_RW_NAME/
mount ${DEV}1 /mnt/CLONEZILLA/
mount ${DEV}2 /mnt/$PART_RW_NAME/


# Extraire ISO dans support

if [[ $DBG_NO_EXTRACT_ISO -ne 1 ]] ; then

 mkdir -p /mnt/ISO
 mount -o loop cz_latest.iso /mnt/ISO/
 rsync -aP /mnt/ISO/ /mnt/CLONEZILLA/
 umount -l /mnt/ISO
 rmdir /mnt/ISO

 rm cz_latest.iso

fi

# Modifier configuration boot (FR et pré-exécuter dhclient)

#~ sed -i -e '/^ *append initrd=/' \
 #~ -e 's/locales=/locales=fr_FR.UTF-8/' \
 #~ -e 's/keyboard-layouts= /keyboard-layouts=fr /' \
 #~ -e 's/enforcing=0 /enforcing=0 ocs_prerun="dhclient" /' \
 #~ /mnt/CLONEZILLA/syslinux/syslinux.cfg
 
sed -i -e '/^ *append initrd=/s/locales=/locales=fr_FR.UTF-8/' \
 -e 's/keyboard-layouts=/keyboard-layouts=fr/' \
 -e 's/enforcing=0 /enforcing=0  ocs_prerun="dhclient" /' \
 /mnt/CLONEZILLA/syslinux/syslinux.cfg

 
# Extraire système interne

mount -t squashfs -o loop /mnt/CLONEZILLA/live/filesystem.squashfs /mnt
mkdir squashfs
rsync -aP /mnt/ squashfs


# Construire script externe

EXTERNE=$( cat <<- EOT
#!/bin/bash

 # Pour réinitialiser supprimer script et exécuter /$INNER_1ST_SCRIPT

 # Script externe auto-généré modifiable
 # Outils : /$INNER_PASSWORD_MODIFIER /$INNER_HOSTNAME_MODIFIER

 # Redéfinir mot de passe ('' pour ne pas redéfinir)
 NEW_PASS=''
 /$INNER_PASSWORD_MODIFIER "\$NEW_PASS" > /dev/null

 # Redéfinir hostname ('' pour ne pas redéfinir)
 NEW_HOSTNAME=''
 /$INNER_HOSTNAME_MODIFIER "\$NEW_HOSTNAME" > /dev/null
   
 # Exporter disques locaux par NBD
 #/usr/lib/live/mount/medium/$SCRIPT_NAME_NBD_EXPORT
 /mnt/$SCRIPT_NAME_NBD_EXPORT
 
 # Personnaliser plus ici ...
 
EOT
)


# Personnaliser système interne

for f in /proc /sys ; do mount -B $f squashfs$f ; done ;
mount -t devpts none squashfs/dev/pts

cat << EOC | \
  externe=$EXTERNE \
  inner_1st_script=$INNER_1ST_SCRIPT \
  inner_password_modifier=$INNER_PASSWORD_MODIFIER \
  inner_hostname_modifier=$INNER_HOSTNAME_MODIFIER \
  part_rw_name=$PART_RW_NAME dns=$DNS pass=$PASS \
  hostname_support=$HOSTNAME_SUPPORT \
  outer_1st_script=$OUTER_1ST_SCRIPT \
  chroot squashfs

 INNER_1ST_SERVICE=run_1st_script.service
 INNER_2ND_SCRIPT=inner_2nd_script.sh
 #inner_password_modifier=password_modifier.sh
 #inner_hostname_modifier=hostname_modifier.sh
 #OUTER_PASSWORD_M_FUNC=m_passwd
 #OUTER_HOSTNAME_M_FUNC=m_hostname
 APP="vim avahi-daemon dnsutils winbind libnss-winbind libnss-mdns \
  memtester nbd-server"


 # Permettre la résolution DNS

 echo \$hostname_support > /etc/hostname
 a=\$(
  echo nameserver \$dns
  cat /etc/resolv.conf \\
   | grep -vE ^[[:space:]]*nameserver[[:space:]]+\$dns[[:space:]]*$
 )
 echo -e "\$a" > /etc/resolv.conf
 
 
 # Installer les outils nécessaires à la session Live

 apt update && apt install -y \$APP
 
 
 # Rendre système accessible par SSH

 a=\$(
  cat /etc/profile \\
   | grep -vE ^[[:space:]]*systemctl[[:space:]]+restart[[:space:]]+ssh
  echo systemctl restart ssh
 )
 echo -e "\$a" > /etc/profile
 a=\$(
  cat /etc/ssh/sshd_config | grep -vE ^[[:space:]]*PermitRootLogin
  echo PermitRootLogin yes
 )
 echo -e "\$a" > /etc/ssh/sshd_config
 systemctl enable ssh
 

 # Créer service qui exécute premier script en Live session (voir [1])
 
 cat << EOF > /etc/systemd/system/\$INNER_1ST_SERVICE

 [Unit]
 Description=Exécute le premier script interne

 [Service]
 Type=Simple
 ExecStart=/bin/bash /\$inner_1st_script
 # ExecStart=/usr/bin/sudo /bin/bash /\$inner_1st_script

 [Install]
 WantedBy=multi-user.target
 
EOF

 chmod 644 /etc/systemd/system/\$INNER_1ST_SERVICE
 systemctl enable \$INNER_1ST_SERVICE


 # Créer premier script d'initialisation qui exécute 2eme script interne

 cat << EOF > \$inner_1st_script
#!/bin/bash
 

 /\$inner_password_modifier '\$pass'
 
 
echo "passwd : /\$inner_password_modifier '\$pass'" >> touched


 ln -s /usr/lib/live/mount/medium /root/support_root
 ln -s /usr/lib/live/mount/medium /home/user/support_root

 ln -s /mnt /root/\$part_rw_name
 ln -s /mnt /home/user/\$part_rw_name
 
 /\$INNER_2ND_SCRIPT
 
 exit


 # Exécuter script interne qui exécute le script externe
 
 part=\\\$(
 
  lsblk -nlo NAME,LABEL | grep \$part_rw_name$ | cut -d ' ' -f 1
  
 )


echo "part_rw_name : \$part_rw_name )" >> touched 
echo "parts : \\\$( lsblk -nlo NAME,LABEL )" >> touched 
echo "part detectée : \\\$part" >> touched
 
 if [[ ! \\\$( mount | grep /dev/\\\$part ) ]] ; then
 
  mount -l /dev/\\\$part /mnt
  
 fi
 
 exit
 
echo "mount opérationnel : \\\$( mount | grep /mnt ; ls -l /mnt )" >> touched

 f=/mnt/\$outer_1st_script
 if [[ ! -f \\\$f \
    || ! \\\$( file \\\$f | grep " text executable$" ) \
    || ! ( \
          \\\$( grep "^#!" \\\$f ) && \
          \\\$( grep "#*\$inner_1st_script" \\\$f ) \
         ) ]] ; then
         
  cp /\$outer_1st_script /mnt/\$outer_1st_script
  chmod +x /mnt/\$outer_1st_script

 fi

echo "outer_1st_script accessible : \\\$( ls -l /mnt )" >> touched


 /mnt/\$outer_1st_script

EOF

 chmod +x \$inner_1st_script
 

 # Créer 2ème script interne qui exécute le script externe

 cat << EOF > \$INNER_2ND_SCRIPT
#!/bin/bash
 
 part=\\\$(
 
  lsblk -nlo NAME,LABEL | grep \$part_rw_name$ | cut -d ' ' -f 1
  
 )

echo "part_rw_name : \$part_rw_name )" >> touched 
echo "parts : \\\$( lsblk -nlo NAME,LABEL )" >> touched 
echo "part detectée : \\\$part" >> touched
 
 if [[ ! \\\$( mount | grep /dev/\\\$part ) ]] ; then
 
  mount -l /dev/\\\$part /mnt
  
 fi
 
echo "mount opérationnel : \\\$( mount | grep /mnt ; ls -l /mnt )" >> touched

 f=/mnt/\$outer_1st_script
 if [[ ! -f \\\$f \
    || ! \\\$( file \\\$f | grep " text executable$" ) \
    || ! ( \
          \\\$( grep "^#!" \\\$f ) && \
          \\\$( grep "#*\$inner_1st_script" \\\$f ) \
         ) ]] ; then
         
  cp /\$outer_1st_script /mnt/\$outer_1st_script
  chmod +x /mnt/\$outer_1st_script

 fi

echo "outer_1st_script accessible : \\\$( ls -l /mnt )" >> touched


 /mnt/\$outer_1st_script

EOF

 chmod +x \$INNER_2ND_SCRIPT


 # Créer sauvegarde du script externe qui sera copié à chaque fois
 
 echo "\$externe" > \$outer_1st_script


 # Créer script modifieur de mot de passe
 
 cat << EOF > \$inner_password_modifier
#!/bin/bash

 if [[ "\\\$1" == "" || "\\\$1" == '' ]] ; then
 
  echo -e "Password modifier\nSyntax :"
  echo "\\\$( basename -- \\\${BASH_SOURCE[0]} ) 'PASSWORD'"
  exit
  
 fi
 
 echo -e "\\\$1\n\\\$1" | passwd user
 echo -e "\\\$1\n\\\$1" | passwd root
 
EOF

 chmod +x \$inner_password_modifier
 
 
 # Créer script modifieur de nom d'hôte
 
 cat << EOF > \$inner_hostname_modifier
#!/bin/bash

 if [[ "\\\$1" == "" || "\\\$1" == '' ]] ; then
 
  echo -e "Hostname modifier\nSyntax :"
  echo "\\\$( basename -- \\\${BASH_SOURCE[0]} ) HOSTNAME"
  exit

 fi
 
 sed -i "s/\$hostname_support/\\\$1/g" /etc/hosts
 echo \\\$1 > /etc/hostname
 hostnamectl set-hostname \\\$1
 systemctl restart avahi-daemon
 
EOF

 chmod +x \$inner_hostname_modifier


 # Créer fonctions plus friendly pour accéder aux scripts internes
 # => ne marche pas !
 
  #~ a=\$(
   #~ cat /etc/bash.bashrc \
   #~ | grep -vE \
    #~ ^[[:space:]]*function[[:space:]]+\$OUTER_PASSWORD_M_FUNC[[:space:]]+
      #~ b="function \$OUTER_PASSWORD_M_FUNC"
   #~ b="\$b { /\$inner_password_modifier '\\\$1' ; }"
   #~ echo "\$b"
  #~ )
 #~ echo "\$a" > /etc/bash.bashrc
  #~ a=\$(
   #~ cat /etc/bash.bashrc \
   #~ | grep -vE \
    #~ ^[[:space:]]*function[[:space:]]+\$OUTER_HOSTNAME_M_FUNC[[:space:]]+
      #~ b="function \$OUTER_HOSTNAME_M_FUNC"
   #~ b="\$b { /\$inner_hostname_modifier \\\$1 ; }"
   #~ echo "\$b"
  #~ )
 #~ echo "\$a" > /etc/bash.bashrc 

EOC

# chroot squashfs ls
# chroot squashfs less inner_1st_script.sh

for f in /proc /sys /dev/pts ; do umount -lf squashfs$f ; done


# Intégrer système interne personnalisé + liste des paquets ajoutés

mksquashfs squashfs filesystem.squashfs -info
umount -l /mnt/
chroot squashfs dpkg-query -W --showformat='${Package}  ${Version}\n' \
 > /mnt/CLONEZILLA/filesystem.manifest
rm -rf squashfs
rm /mnt/CLONEZILLA/live/filesystem.squashfs
rsync -P filesystem.squashfs /mnt/CLONEZILLA/live \
 && rm filesystem.squashfs
chmod 444 /mnt/CLONEZILLA/live/filesystem.squashfs


## Script à exécuter en 1er ; peut être modifié en LIVE session


## TODO : récupérer le contenu du script depuis variable

#cat << EOF > /mnt/$PART_RW_NAME/$OUTER_1ST_SCRIPT
##!/bin/bash

 ## touch /touched
 ## => OK


 ## Rédéfinir
 

 ## Host Name pour multiple Clonezilla distants (defaut : $HOSTNAME_SUPPORT)
 ## Laisser vide pour ne pas redéfinir

 #NEW_HOSTNAME=


 ## Mot de passe de session (defaut : '$PASS')
 ## Laisser '' pour ne pas redéfinir
 ## Nota : 'PASS' protège PASS d'une expansion non voulue de * ou ! etc.
 
 #NEW_PASS=''


 #if [[ ! -z \$NEW_HOSTNAME ]] ; then
 
  #sed -i "s/$HOSTNAME_SUPPORT/\$NEW_HOSTNAME/g" /etc/hosts
  #echo \$NEW_HOSTNAME > /etc/hostname
  #hostnamectl set-hostname \$NEW_HOSTNAME
  #systemctl restart avahi-daemon
  
 #fi

 #if [[ ! \$NEW_PASS == '' ]] ; then
 
  ## echo \$NEW_PASS > /touched
  ##echo -e "\$NEW_PASS\n\$NEW_PASS" | passwd user
  ##echo -e "\$NEW_PASS\n\$NEW_PASS" | passwd root
  ## echo retour passwd : $?
  
  ## passwdm '\$NEW_PASS'
  
 #fi

 #/usr/lib/live/mount/medium/$SCRIPT_NAME_NBD_EXPORT
 #/mnt/$SCRIPT_NAME_NBD_EXPORT

#EOF

#chmod +x /mnt/$PART_RW_NAME/$OUTER_1ST_SCRIPT


# TODO : quand opérationnel intégrer ce script directement dans le chroot, on y fera appel avec un m_xxx

# Exporter disques locaux
# Nota : heredoc avec variables non expandées

#cat << 'EOF' > /mnt/CLONEZILLA/$SCRIPT_NAME_NBD_EXPORT
cat << 'EOF' > /mnt/$PART_RW_NAME/$SCRIPT_NAME_NBD_EXPORT
#!/bin/bash

 if [[ $( netstat -atnp | grep nbd-server | grep ESTABLISHED ) ]] ; then
 
  msg="Erreur : les disques sont déjà importés à distance"
  msg="$msg\nVoici la liste :"
  msg="$msg\n$( netstat -atnp | grep nbd-server | grep ESTABLISHED )"
  echo -e "$msg"
  
  exit 1
  
 fi

 modprobe -r nbd
 pkill nbd-server


 # Configurer disques importables à distance
 
 (
 
  echo -e "[generic]\nuser=root\ngroup=root\nallowlist=true"
  
  for e in $(  
    lsblk -nd -o NAME,SIZE | awk '$2 != "0B" { print $1 "|" $2 }'   
  ) ; do
  
   dev=$( echo $e | cut -d '|' -f 1 )
   
   id=$(
    find /dev -type l \
         -exec bash -c 'l={} ; echo $l $( readlink $l )' \; \
    | grep /by-id/.*$dev$ | sort -r | tail -n 1 | cut -d ' ' -f 1 \
    | rev | cut -d '/' -f 1 | rev
   )
   
   size=$( echo $e | cut -d '|' -f 2 )
   echo -e "[$dev@$id@$size]\nexportname=/dev/$dev"
  
  done

 ) > /nbd.conf
 
 modprobe nbd

 # Je ne sais pas pourquoi mais il faut le démarrer avec -d (daemon ?)
 nbd-server -C /nbd.conf -d &

EOF

#chmod +x /mnt/CLONEZILLA/$SCRIPT_NAME_NBD_EXPORT
chmod +x /mnt/$PART_RW_NAME/$SCRIPT_NAME_NBD_EXPORT


# Rendre support bootable

if [[ $DBG_NO_MAKE_BOOTABLE -ne 1 ]] ; then

  bash /mnt/CLONEZILLA/utils/linux/makeboot.sh \
   -L $HOSTNAME_SUPPORT -b ${DEV}1

fi


# Démonter support

#umount -l /mnt/CLONEZILLA /mnt/$PART_RW_NAME
#rmdir /mnt/CLONEZILLA /mnt/$PART_RW_NAME
umount -l /mnt/*
rmdir /mnt/*
```
