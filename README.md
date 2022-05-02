# nbd_remote_disk

Brouillon en attendant une vraie doc

# SERVER (hôte qui partage un disque local)
	root@goku:~# apt install nbd-serveur
	root@goku:~# modprobe nbd

	# Voir aussi "Serveur : auto configuration"
	root@CZ-LIVE:~# cat < EOF > /etc/nbd-server/config
	[generic]
	# If you want to run everything as root rather than the nbd user, you
	# may either say "root" in the two following lines, or remove them
	# altogether. Do not remove the [generic] section, however.
	#       user = nbd
	#       group = nbd

	# Je n'ai pas encore creusé mais si on laisse nbd en user/group
	# J'ai l'erreur :
	# Negotiation: ..Error: Connection not allowed by server policy. Server said: Access denied by server configuration
	#
			user = root
			group = root

	# On peut redéfinir le port (par défaut : 10809)
	# Dans ce cas les commandes clientes seront modifiées :
	# >	nbd-client -l <server> <port>
	# >	nbd-client <server> <port> -N <nom export> <attached_port>
	#       port = 9999

			includedir = /etc/nbd-server/conf.d

	# Permet au client de lister les exports
			allowlist = true

	# What follows are export definitions. You may create as much of them as
	# you want, but the section header has to be unique.
	[export]
			exportname = /dev/sda

	# limiter les clients
	#       listenaddr = 192.168.0.52

	# test 2eme export
	[export2]
			exportname = /dev/sda
	EOF

	# Exporter depuis configuration par défaut : /etc/nbd-server/config
	root@CZ-LIVE:/dev# nbd-server
	# ou depuis autre configuration
	root@CZ-LIVE:/dev# nbd-server -C /chemin/fichier/config
	# => rien ne s'affiche, c'est normal !

	# Si aucun processus, il y a surement une erreur de configuration
	root@CZ-LIVE:/dev# ps aux | grep nbd
	root        2578  0.0  0.0  12060  4748 ?        Ss   10:19   0:00 nbd-server
	root        2603  0.0  0.0  25256  2384 pts/0    S+   10:28   0:00 grep nbd

	root@CZ-LIVE:~# netstat -tlnp | grep nbd
	tcp6       0      0 :::10809                :::*                    LISTEN      2759/nbd-serve

	# Pour arrêter le processus
	# TODO : faire de la manière propre ...
	root@CZ-LIVE:/dev# kill 2578



# CLIENT (hôte auquel on va attacher le disque à distance sur SERVER)

	root@goku:~# apt install nbd-client

	# Nota : dans les versions actuelles le module s'appelle nbd et PAS nbd-client
	root@goku:~# lsmod | grep nbd
	root@goku:~# modprobe nbd
	root@goku:~# lsmod | grep nbd
	nbd                    53248  3

	root@goku:~# mount /dev/nbd0p1 /mnt/

	# Lister les exports du serveur
	root@goku:~# nbd-client -l CZ-LIVE.local
	Negotiation: ..
	export
	export2

	root@goku:~# nbd-client CZ-LIVE.local -N export /dev/nbd0
	Negotiation: ..size = 20480MB
	Connected /dev/nbd0

	# <en parallèle>

	root@goku:~# dmesg -w
	...
	[12262.982449]  nbd0: p1 p2 < p5 >
	[12308.925234] EXT4-fs (nbd0p1): mounted filesystem with ordered data mode. Opts: (null)

	# <en parallèle/>

	root@goku:~# lsblk /dev/nbd0
	NAME     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
	nbd0      43:0    0   20G  0 disk 
	├─nbd0p1  43:1    0   19G  0 part /mnt
	├─nbd0p2  43:2    0    1K  0 part 
	└─nbd0p5  43:5    0  975M  0 part

	root@goku:~# mount /dev/nbd0p1 /mnt/

	root@goku:~# ls /mnt
	bin  boot  dev	etc  home  initrd.img  initrd.img.old  lib  lib32  lib64  libx32  lost+found  media  mnt  opt  proc  root  run	sbin  srv  sys	tmp  usr  var  vmlinuz	vmlinuz.old

	# Pour détacher
	root@goku:~# nbd-client -d /dev/nbd0
	
# Serveur : auto configuration
Exporte automatiquement tous les disques locaux du serveur

Avant de tuer tout processus nbd-server, s'assurer qu'il n'y a plus aucune connexion aux disques à distance :

	root@server:~# netstat -atnp | grep nbd
	tcp6       0      0 :::10809                :::*                    LISTEN      125928/nbd-server   
	tcp6       0      0 192.168.0.15:10809      192.168.0.52:48736      ESTABLISHED 125931/nbd-server

=> ici une connexion depuis **192.168.0.15** (```umount -l /chemin/montage``` suivi de ```nbd-client -d /chemin/disque_local_nbd/lie/disque_distant```)

S'il y en a plus, on peut tuer les éventuels processus en cours ... :

	root@server:~# pkill nbd-server

... et auto-configurer le server :

	root@sever/# ( \
	 echo -e "[generic]\nuser=root\ngroup=root\nallowlist=true"
	 for e in $( \
	   lsblk -nd -o NAME,SIZE | awk '$2 != "0B" { print $1 "|" $2 }' \
	 ) ; do
	  dev=$( echo $e | cut -d '|' -f 1 )
	  id=$( \
	   find /dev -type l \
		-exec bash -c "l={} ; echo \$l \$( readlink \$l )" \; \
		 | grep /by-id/.*$dev$ | cut -d ' ' -f 1 \
		 | rev | cut -d '/' -f 1 | rev )
	  size=$( echo $e | cut -d '|' -f 2 )
	  echo -e "[$dev@$id@$size]\nexportname=/dev/$dev"
	 done
	) > nbd.conf

Depuis le client la liste retournée par afficher l'ID de disque et la taille :

	root@client:~# 	nbd-client -l SERVER
	Negotiation: ..
	loop0@@484,2M
	sda@ata-QEMU_HARDDISK_QM00005@20G
	sdb@usb-QEMU_QEMU_HARDDISK_1-0000:00:05.7-4-0:0@1G

Et pour attacher :

	root@client:~# nbd-client SERVER -N sda@ata-QEMU_HARDDISK_QM00005@20G /dev/nbd0
	Negotiation: ..size = 20480MB
	Connected /dev/nbd0



