# coreboot_guide
Coreboot and me_cleaner: Free Your BIOS ( updated and translated https://connect.ed-diamond.com/GNU-Linux-Magazine/GLMF-220/Coreboot-et-me_cleaner-liberez-votre-BIOS)
Cet article sʼinscrit dans une volonté de libérer du matériel récent à bas niveau. Actuellement, seul libreboot (distribution de coreboot) permet de complètement enlever le « Mangement Engine (ME) » dʼIntel et autres blobs propriétaires. Il existe cependant une possibilité de neutraliser le ME avec me_cleaner et de le réduire à ses fonctions les plus primaires.
Le matériel le plus avancé compatible avec Libreboot en Intel date de 2008 (Lenovo X200/T) et 2010 côté AMD (Asus KPGE-D16), donc du matériel plutôt ancien bien quʼil tienne encore la route pour un usage bureautique ou multimédia simple. Il existe cependant une possibilité de neutraliser le ME avec me_cleaner et de le réduire à ses fonctions les plus primaires. Nous verrons ici comment utiliser ce logiciel non pas sur un BIOS standard (ce qui est possible), mais avec Coreboot et SeaBIOS Lʼobjectif étant dʼavoir une nouvelle image de BIOS libre et avec le moins de blobs propriétaires possible. Je précise toutefois que la manipulation a été faite avec un Lenovo X230. Je nʼai pas dʼaction chez eux, mais cʼest une marque dont les ordinateurs portables sont particulièrement bien supportés par la communauté Coreboot/Libreboot et pour laquelle nous trouvons beaucoup de drivers libres et une bonne documentation technique. Tout ce qui est présenté dans cet article est donc valable pour la plupart des Thinkpad récents.
Le problème que lʼon rencontre sur le matériel plus récent cʼest quʼil est impossible de complètement retirer le ME sinon lʼordinateur sʼéteint au bout de 30 minutes par mesure de sécurité imposée par le logiciel privateur dʼIntel. Il existe cependant une solution : me_cleaner[1] et Coreboot[2]. Ce petit logiciel permet de neutraliser le ME et, utilisé avec Coreboot, réduire sa taille considérablement. Nous verrons les détails dans quelques instants.
Pour rappel, Coreboot (anciennement LinuxBIOS) est un projet de logiciel libre dʼamorçage créé en 1999. Il vise à remplacer les BIOS propriétaires trouvés dans la plupart des ordinateurs par un système dont la fonction exclusive est de charger un système dʼexploitation moderne à 32 ou 64 bits.
Il est écrit en C et en assembleur X86. me_cleaner est un script écrit en Python qui peut modifier le ME dʼIntel dans une image de BIOS compilée dont la finalité est de réduire à son strict minimum la capacité de ce firmware à interagir avec le système. Ci-après une petite matrice de risques quant aux parties non libres contenues dans un BIOS :

Coreboot remplace généralement les parties VGA, EC et ne touche pas au microcode du CPU. Il intègre par défaut le ME dʼorigine, mais peut prendre un ME neutralisé et réduit via me_cleaner comme nous le verrons ultérieurement.
1. Quelques éclaircissements
1.1 Quʼest-ce que le Management Engine ?
Le moteur de gestion Intel (ME « Management Engine » en anglais) est un sous-système autonome qui a été incorporé dans presque tous les chipsets de processeur dʼIntel depuis 2008. Le sous- système consiste principalement en un microprogramme propriétaire fonctionnant sur un microprocesseur distinct qui exécute des tâches pendant le démarrage, pendant que lʼordinateur est en cours dʼexécution et pendant quʼil est en veille. Tant que le chipset ou le SoC est connecté au courant (via la batterie ou lʼalimentation), il continue à fonctionner même lorsque le système est éteint. Intel affirme que le ME est nécessaire pour fournir une performance complète. Ses fonctionnements exacts sont non documentés et son code est obscurci en utilisant des tables confidentielles de Huffman stockées directement dans le matériel, de sorte que le firmware ne contient pas les informations nécessaires pour décoder son contenu. De la retro ingénierie bas niveau a cependant permis de comprendre une bonne partie des mécanismes de ME. Le principal concurrent dʼIntel, AMD, a incorporé lʼéquivalent, AMD Secure Technology (anciennement appelé Platform Security Processor), dans la quasi-totalité des processeurs post-2013.
Le moteur de gestion est souvent confondu avec Intel AMT. AMT est basé sur le ME, mais seulement disponible sur les processeurs avec la technologie vPro. AMT permet aux propriétaires dʼadministrer à distance leur ordinateur, comme lʼallumer ou lʼéteindre et réinstaller le système dʼexploitation.
Cependant, le ME lui-même est intégré dans tous les chipsets Intel depuis 2008, pas seulement ceux avec AMT. Tandis que AMT peut être non provisionné par le propriétaire, il nʼy a aucune manière officielle et documentée de désactiver le ME.
LʼElectronic Frontier Foundation (EFF) et lʼexpert en sécurité Damien Zammit accusent le ME dʼêtre une porte dérobée et un problème de confidentialité. Zammit déclare que le ME a un accès complet à la mémoire (sans que le CPU parent en ait connaissance), a un accès complet à la pile TCP/IP et peut envoyer et recevoir des paquets réseau indépendamment du système dʼexploitation, contournant ainsi son pare feu. Intel affirme quʼil « ne remet pas en question les portes dans ses produits » et que ses produits « ne permettent pas à Intel de contrôler ou dʼaccéder aux systèmes informatiques sans lʼautorisation explicite de lʼutilisateur final ».
Plusieurs faiblesses ont été trouvées dans le ME. Le 1er mai 2017, Intel a confirmé lʼexistence dʼun bogue Remote Elevation of Privilege (SA-00075) dans sa technologie de gestion. Chaque plateforme Intel dotée de la technologie de gestion standard, de gestion active ou de petites technologies Intel fournies, de Nehalem en 2008 à Kaby Lake en 2017, dispose dʼune faille de sécurité exploitable à distance dans le ME. Plusieurs façons de désactiver le ME sans autorisation qui pourraient permettre aux fonctions de ME dʼêtre sabotées ont été trouvées. Dʼimportantes failles de sécurité supplémentaires dans le ME affectant un très grand nombre dʼordinateurs intégrant le micrologiciel ME, Trusted Execution Engine (TXE) et Server Platform Services (SPS), de Skylake en 2015 à Coffee Lake en 2017, ont été confirmées par Intel le 20 novembre 2017 (SA-00086). Contrairement à SA-00075, ce bug est même présent si AMT est absent, non provisionné ou si le ME a été « désactivé » par lʼune des méthodes non officielles connues.

1.2 Pourquoi le ME dʼIntel est-il si nuisible et comment est-il fait ?
Le Management Engine (ME) dʼIntel est un logiciel propriétaire opaque et dont la fonction exacte nʼest pas vraiment connue. Il a un accès illimité à plusieurs branches de lʼordinateur (réseau,
mémoire, disque dur…) et il est complètement transparent vis-à-vis du reste du système (donc ce quʼil fait nʼest pas détectable depuis le système dʼexploitation). Entre autres choses néfastes, le ME
peut contrôler à distance presque tout lʼordinateur, ce qui constitue une très importante faille de sécurité (voir figure 1).
Lʼobjectif du me_cleaner est ici de désactiver le ME après la phase de boot et limiter le ME a la stricte initialisation du matériel. Ainsi aucun accès mémoire, disque ou réseau ne peut avoir lieu après
lʼamorçage du système et donc aucun accès aux données privées de lʼutilisateur final. Il y a par ailleurs une fonction qui permet à me_cleaner de réduire lʼespace occupé par le ME, et ainsi
augmenter celui de coreboot (et SeaBIOS) permettant de rajouter des fonctionnalités (nous verrons cela dans une autre partie de l'article).

Note
Qu'est-ce que SeaBIOS ?
SeaBIOS est une implémentation open source dʼun BIOS x86 16 bits, servant de microprogramme disponible gratuitement pour les systèmes x86. Visant la compatibilité, il prend en charge les fonctionnalités standards du BIOS et les interfaces dʼappel qui sont implémentées par un BIOS x86 propriétaire typique. SeaBIOS peut être utilisé comme payload par coreboot, ou peut être utilisé directement dans des émulateurs tels que QEMU et Bochs. Initialement, SeaBIOS était basé sur lʼimplémentation du BIOS open source incluse avec lʼémulateur Bochs. Le projet a été créé avec lʼintention de permettre une utilisation native sur le matériel x86 et dʼêtre basé sur une implémentation de code source interne améliorée et plus facilement extensible.
SeaBIOS nʼest pas le seul payload que lʼon peut utiliser avec Coreboot. Il est aussi possible dʼinstaller un GRUB(1 ou 2), un micro-linux… Il existe plusieurs alternatives que lʼon trouve facilement sur le Web. Cependant cʼest clairement celui qui sʼinstalle et se configure le plus simplement. On peut également le modifier depuis le système dʼexploitation avec le logiciel nvramtools.

1.3 Comment est fait un BIOS ?
Étant donné que le test a été fait sur un Lenovo X230, je vais expliquer principalement comment fonctionne ce BIOS. Il a la particularité dʼavoir deux puces physiques pour une virtuelle mais, sur le principe, tous les BIOS x86 fonctionnent sur le modèle de la puce virtuelle (voir figure 2). Dans notre cas, nous avons deux puces flash SPI qui se cachent sous le plastique noir, étiquetées « SPI1 » et « SPI2 ». Visuellement, le premier est 4 Mio et contient le BIOS et le vecteur de réinitialisation (ce qui permet de faire fonctionner le bouton « Reset »). Celui du bas est de 8 Mio, il contient lʼIntel Management Engine (ME), lʼamorceur de réseau (GbE) et le descripteur flash. Les deux puces sont concaténées dans une puce virtuelle de 12Mio. On parle de régions de BIOS.

1.4 Comment agissent Coreboot et me_cleaner sur cette structure ?
Si lʼon reprend la structure (sans parler pour le moment dʼespace occupé), lʼIntel Flash Descriptor (IFD) est modifié par me_cleaner pour compenser la perte dʼespace du ME ; le BIOS, sans blobs VGA, est remplacé par coreboot + SeaBIOS (nous verrons plus tard ce que cʼest) et le GbE reste inchangé (voir figure 3). Nous verrons en dernière partie comment lʼespace occupé par lʼensemble des régions change avant et après les différentes étapes.

2. Préparer la compilation
2.1 Prérequis logiciels et matériels
Au niveau du logiciel, nous avons besoin dʼun ordinateur hôte avec un système GNU/Linux dessus (jʼai utilisé Ubuntu 18.04 pour ce test, mais lʼopération fonctionne sur nʼimporte quelle distribution).
Pensez à être le plus à jour possible pour éviter des problèmes de compatibilité de matériel, de compilation ou autre. Pour manipuler les images de BIOS, vous aurez besoin de flashrom installé dans sa dernière version, git et GCC à jour également.
Pour le matériel, plusieurs solutions sont possibles et tout dépend du type de BIOS à flasher. Pour les cas les plus courants, les puces de BIOS sont à 8 ou 16 pins. Je conseille donc un programmateur de puce type CH341A (très abordable et simple à utiliser). Vous avez tout un tas dʼautres programmateurs disponibles sur le marché, mais généralement assez chers et complexes à utiliser. 
Pour relier le programmateur au BIOS, il faudra nécessairement un clip SOIC 8 ou 16 en fonction de la taille de la puce et son câble pour le relier au programmateur. Je vous conseille de prendre la plus petite longueur possible pour éviter des problèmes lors des opérations sur le BIOS. Il faut bien entendu tout lʼoutillage traditionnel pour démonter un PC fixe ou portable (tournevis, clips en tous genres, etc.).
2.2 Préparation de lʼhôte linux
La première étape consistera donc à installer sur lʼordinateur qui permettra de compiler Coreboot lʼensemble des logiciels et codes sources nécessaires. On va donc procéder comme suit :
Installation de flashrom, git et GCC : 
```
$ sudo apt install flashrom bison build-essential curl flex git gnat libncurses5-dev libssl-dev m4 zlib1g-dev pkg-config wget
```
On va ensuite récupérer les sources de Coreboot et me_cleaner puis compiler les différents outils (placez-vous à la base du home pour faire au plus simple). Il faut également créer 3 dossiers dont mainboard qui est générique puis un dossier selon la marque de la carte mère (pour le test Lenovo) et le modèle (pour le test X230). Des exemples sont disponibles dans la documentation officielle de Coreboot.
```
$ cd ~/
$ git clone https://review.coreboot.org/coreboot
$ cd coreboot
$ git submodule update --init --checkout
$ make crossgcc-i386 CPUS=$(nproc)
$ cd util/ifdtool
$ make -j4 && sudo make install
$ cd ~/coreboot
$ mkdir -p 3rdparty/blobs/mainboard/lenovo/x230/
```
2.3 Préparation de la cible (lʼordinateur à flasher)
La préparation a deux grandes phases : rendre le BIOS accessible (donc démonter le PC pour avoir accès au BIOS sur la carte mère comme on peut le voir en figure 4) et installer le clip (figure 5) sur le BIOS (attention au sens, il faut placer le numéro 1 du pin au même endroit sur le programmateur que sur le BIOS). Ensuite, on relie le clip au programmateur et le programmateur au PC (voir figure 6).
Pensez à bien sauvegarder plusieurs fois lʼimage dʼorigine du BIOS pour revenir en arrière en cas dʼerreur avec flashrom[3] et la commande suivante (à adapter selon le type de BIOS) :
```
$ sudo flashrom -p ch341a_spi -r backup.rom
```
À noter que dans le cas où il y a deux puces physiques pour une puce virtuelle, il faut sauvegarder les deux et les concaténer en une seule complete.rom à lʼaide la commande cat pour le reste des
opérations.
```
$ sudo flashrom -p ch341a_spi -r 8MiB.rom
$ sudo flashrom -p ch341a_spi -r 4MiB.rom
$ cat 8MiB.rom 4MiB.rom > complete.rom
```
2.4 Récupérer les données pour créer une rom Coreboot et neutraliser le ME
Placez-vous là où se trouve lʼimage complete.rom (généralement ~/). Il faut maintenant extraire toutes les régions du BIOS dʼorigine pour récupérer les blobs tels que le ME ou le GbE. Exécutez donc la commande suivante :
```
$ ifdtool -x complete.rom
```
Vous devrez trouver les fichiers suivants (rappelez-vous la première partie sur la structure dʼun BIOS) :
• flashregion_0_flashdescriptor.bin ;
• flashregion_1_BIOS.bin ;
• flashregion_2_intel_me.bin ;
• flashregion_3_gbe.bin.
Copiez ensuite flashregion_3_gbe.bin vers le dossier 3rdparty/blobs/mainboard/lenovo/x230 en le renommant comme suit :
```
$ cp flashregion_3_gbe.bin 3rdparty/blobs/mainboard/lenovo/x230/gbe.bin
```
Il faut maintenant neutraliser et réduire le ME à lʼaide de me_cleaner. Ce procédé va réduire la taille de la région du ME en la divisant par 5 par rapport à lʼoriginale. Il suffit de lancer la commande suivante :
```
$ python3 ~/coreboot/util/me_cleaner/me_cleaner.py -S -r -t -d -O out.bin -D ifd_shrinked.bin -M me_shrinked.bin ./complete.rom
```
Nous prêterons plus dʼattention à cette commande dans une prochaine partie pour expliquer en détail le fonctionnement de me_cleaner et son effet sur la ROM finale. Ce quʼil faut savoir pour le moment cʼest que me_shrinked.bin correspond au flashregion_2_intel_me.bin de la ROM dʼorigine, mais avec un ME neutralisé et une taille réduite (par 5 environ) et que ifd_sjrinked.bin correspond au nouveau descripteur de flash (région ) qui indique comment sont placés les régions les unes par rapport aux autres. Au final, nous avons maintenant 3 nouveaux fichiers :
• out.bin qui est inutile ;
• ifd_shrinked.bin, le nouveau descripteur ;
• me_shrinked.bin, le nouveau ME neutralisé et réduit.

Il faut maintenant copier le nouveau ME et le nouveau descripteur dans le dossier des blobs :
```
$ cp ifd_shrinked.bin 3rdparty/blobs/mainboard/lenovo/x230/descriptor.bin
$ cp me_shrinked.bin 3rdparty/blobs/mainboard/lenovo/x230/me.bin
```
À ce stade des opérations, nous avons tous les éléments nécessaires pour construire une nouvelle ROM basée sur Coreboot et SeaBIOS.

3. Compiler une nouvelle ROM Coreboot et flasher le Thinkpad
3.1 Compilation de la ROM Coreboot
Nous allons maintenant très simplement compiler une nouvelle ROM Coreboot. On commence par se rendre dans le dossier coreboot :
```
$ cd ~/coreboot
```
Étant donné que tous les blobs ont été copiés, nous pouvons rentrer dans le menu qui permet deconfigurer la future ROM Coreboot et créer un fichier.conf qui servira à donner les indications de compilation. Depuis la version 4.8 de Coreboot, SeaBIOS se télécharge, se configure et se compile dans la ROM finale automatiquement.
```
$ make nconfig
```
Une fenêtre sʼouvre dans le terminal, il faudra y naviguer avec les flèches directionnelles du clavier et utiliser la barre dʼespace pour sélectionner une valeur.

Dans le menu General Setup : Set « Set CMOS for configuration values ».
Dans le menu Payloads : Set « Add a payload » → SeaBIOS → git version.
Dans le menu Mainboard : Set « Mainboard Vendor » to “Lenovo” et
Set « Mainboard model » to « X230 ».
Dans le menu Chipset (les chemins complets s'ajoutent automatiquement):
Set « Add Intel descriptor.bin file »,
Set « Add Intel ME firmware » et
Set « Add Gigabite Ethernet firmware ».
Dans le menu Devices : Set « Use native graphics initialization ».
Voilà, la configuration est terminée, il faut maintenant compiler la ROM :
```
$ make CPUS=$(nproc)
```
La ROM est générée dans ~/coreboot/builds/coreboot.rom.

3.2 Flasher le portable avec la nouvelle ROM
Note
Flasher un BIOS
Attention tout de même ! La manipulation est risquée et peut rendre lʼordinateur inopérant. 
Construire et remplacer un BIOS nʼest pas une opération simple. Cela demande du temps, du courage et beaucoup de patience. Plusieurs étapes de vérification sont nécessaires, tout comme un matériel adapté. Évitez les soudures ou tout chose qui peut altérer physiquement le matériel.
Avant de flasher de façon hardware, il faut préparer les deux ROM qui iront sur la puce de 4 Mio et celle de 8 Mio. Pour rappel (voir partie 2), il y a deux puces dans un ordre précis 8 Mio dʼabord et 4 Mio ensuite. Il faut donc maintenant séparer la ROM récemment créée. Pour cela, on va lancer les commandes suivantes :
```
$ dd of=new_top.rom bs=1M if=~/coreboot/builds/coreboot.rom skip=8
$ dd of=`ew_bottom.rom bs=1M if=~/coreboot/builds/coreboot.rom count=8
```
Lʼargument bs (block size) 1M veut dire que lʼon fait des pas de 1 Mio, lʼargument skip=8 signifie que lʼon saute les 8 premiers blocs de 1M donc que lʼon ne prend que la partie finale de 4 Mio, lʼargument
count est lʼinverse, on ne prend que les 8 premiers blocs de 1 Mio. Au final, nous avons donc deux images, une de 4 Mio et une de 8 Mio. Comme les deux images sont ensuite assemblées dans une image virtuelle de 12 Mio peu importe la répartition des régions au sein des deux puces dès lʼinstant que le descripteur est dans la puce de 8 Mio (donc au début). On peut maintenant flasher le BIOS avec la même commande pour lʼextraction ou presque (avec la même contrainte matérielle que dans la partie précédente). Vous pouvez ensuite redémarrer le PC.
```
$ sudo flashrom -p ch341a_spi -r new_bottom.rom -c 
$ sudo flashrom -p ch341a_spi -r new_top.rom
```
4 Résultats de lʼopération et explications
Voyons maintenant ce que lʼon a dans la nouvelle ROM par rapport à lʼancienne avec un simple ls -al (jʼai fait un idftools -x sur la ROM complet.rom dʼorigine et sur coreboot.rom qui est la nouvelle image de BIOS (avec la région du ME réduit et neutralisée par me_cleaner)) :
```
ThinkPad-X230:~/coreboot/local_BIOS$ ll *.bin
— rw-r--r-- 1 robin robin 12472320 juil. 9 21:33 BIOS_local.bin
— rw-r--r-- 1 robin robin 7340032 juil. 9 21:39 BIOS_origine.bin
— rw-r--r-- 1 robin robin 98304 juil. 9 21:33 me_local.bin
— rw-r--r-- 1 robin robin 5230592 juil. 9 21:39 me_origine.bin
```
On remarque tout de suite deux choses : la région ME et la région BIOS ont changé de taille. En effet, me_cleaner a réduit la taille du ME pour ne garder que les fonctions essentielles à lʼinitialisation des composants. La taille est passée de 5,230592 Mio à 0,98304 Mio, soit une division par plus de 5 ! Lʼespace libéré est ensuite récupéré par le BIOS dʼoù la génération par me_cleaner de 2 binaires au lieu dʼun. Le GbE et le descripteur restent inchangés en taille.
Comme nous avions vu au début de ce document, la structure initiale du BIOS correspond à la figure

7.
Lʼutilisation de la commande de neutralisation et de réduction du ME a affecté la structure du ME et donc contraint de modifier le descripteur pour garder la cohérence de la ROM et permettre de flasher à nouveau la puce sans incidence (voir figure 8).

Conclusion
En conclusion, nous avons vu quʼil est possible et assez simple de remplacer son BIOS par une alternative libre et limiter les failles de sécurité sur les plateformes Intel en attendant de meilleurs jours. Je vous invite à regarder les différents liens en référence et lire les documents annexes pour mieux comprendre les enjeux de sécurité à bas niveau, comment les failles sont exploitées et comment aller encore plus loin dans la sécurisation de votre ordinateur avec Coreboot.

Références
[1] Documentation officielle de Coreboot : https://coreboot.org/users.html (https://coreboot.org/users.html)
[2] Page officielle de me_cleaner : https://github.com/corna/me_cleaner (https://github.com/corna/me_cleaner)
[3] Page officielle du matériel supporté par flashrom : https://www.flashrom.org/Supported_hardware (https://www.flashrom.org/Supported_hardware).

Pour aller plus loin
La présentation « Intel ME Secrets » de Igor Skochinsky au RECON de Montréal (Canada) en 2014 : https://recon.cx/2014/slides/Recon%202014 %20Skochinsky.pdf (https://recon.cx/2014/slides/Recon%202014 %20Skochinsky.pdf)
La présentation « Intel ME : The Way of the Static Analysis » par lʼéquipe de recherche de Positive Technologies, 2017, Heidelberg (Allemagne) :
https://www.troopers.de/downloads/troopers17/TR17_ME11_Static.pdf (https://www.troopers.de/downloads/troopers17/TR17_ME11_Static.pdf)
Article sur le blog de la FSF « The Intel Management Engine: an attack on computer users' freedom »avec la contribution de Denis GNUtoo Carikli, Molly de Blanc, du 10 janvier 2018 : https://www.fsf.org/blogs/sysadmin/the-management-engine-an-attack-on-computer-users-freedom (https://www.fsf.org/blogs/sysadmin/the-management-engine-an-attack-on-computer-users-freedom)
Site officiel des développeurs de Heads, une distribution ultra-sécurisée de Coreboot : http://osresearch.net/ (http://osresearch.net/)

