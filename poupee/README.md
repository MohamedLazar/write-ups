
# Challenge Poupées russes

Pièce jointe du challenge :

```
logo.png
```
---
![](images/logo.png)

---
La solution du challenge sera réalisée sous linux (ubuntu).

## Solution

Dans un premier temps, on récupère l'image du challenge :

```
wget -c https://cyber-stuff.net/logo.png
```
On fait un strings dessus mais on ne trouve rien d'intéressant ( pour l'instant ), seulement un certificat d'authenticité.

---

On va analyser l'image avec un outil spécifique pour les fichiers PNG -> ZSTEG.
**Zsteg** permet de détecter des **données cachées** dans un fichier **PNG** ou **BMP**.

Installation de l'outil ( nécessite ruby -> apt install ruby ) :

```
gem install zsteg
```

On lance l'analyse sur l'image :

```
zsteg -a logo.png
```
L'option **-a** nous permet d'utiliser toutes les méthodes connues pour cacher de la donnée.

Première information, on remarque qu'un fichier **7Zip** est caché dans l'image. On retrouve le certificat d'authenticité au début de l'analyse, puis on trouve des données intéressantes :
```
"LE MOT DE PASSE DU FICHIER 7Z (SI VOUS LAVEZ TROUVE !) EST LE SUIVANT : 
A/H/MNgB7WkYbAQEfkJtIiCSvpUbDrGqxQUiD/cHRLI= MAIS ATTENTION, IL A ETE CHIFFRE AVEC CETTE CLEF PUBLIQUE : 
-----BEGIN PUBLIC KEY----- MDswDQYJKoZIhvcNAQEBBQADKgAwJwIgBupRsL/zDsEOlJQpFL"
```
On trouve un texte chiffré ainsi que la clé publique qui a été utilisée. On remarque que la clé n'est pas entière, mais l'option **-a** que nous avons passé va nous aider.
On poursuit le résultat de l'analyse :
```
"M =ILRHc/DiUQxqGrDbUpvSCiItJkfEQAbYkW7BgNM/H/A : TNAVIUS EL TSE )! EVUORT ZEVAL SUOV IS
( Z7 REIHCIF UD ESSAP ED TOM EL -----YEK CILBUP DNE----- 
==QAAEwACkPaQ32RSMQi x+kugwPtFVnP6KLFpQJlOEsDz/LsRpuBgIwJwAgKDAQBBEQANcvhIZoKJYQDwsDM 
-----YEK CILBUP NIGEB-----"
```
On obtient la clé en entier et le texte chiffré dans le sens inverse. On retourne les caractères et on peut commencer à retrouver la clé privée.

On enlève l'espace entre le i et le x et on retourne les caractères :
```
echo -----YEK CILBUP DNE----- ==QAAEwACkPaQ32RSMQix+kugwPtFVnP6KLFpQJlOEsDz/LsRpuBgIwJwAgKDAQBBEQANcvhIZoKJYQDwsDM -----YEK CILBUP NIGEB----- | rev
```
```
-----BEGIN PUBLIC KEY----- MDswDQYJKoZIhvcNAQEBBQADKgAwJwIgBupRsL/zDsEOlJQpFLK6PnVFtPwguk+xiQMSR23QaPkCAwEAAQ== -----END PUBLIC KEY-----
```
On stocke la clé publique dans un fichier **pubkey.pem** :
 
```
-----BEGIN PUBLIC KEY----- 
MDswDQYJKoZIhvcNAQEBBQADKgAwJwIgBupRsL/zDsEOlJQpFLK6PnVFtPwguk+xiQMSR23QaPkCAwEAAQ== 
-----END PUBLIC KEY-----
```


Pour trouver la clé privée, on a besoin de plusieurs informations :
* **N** = valeur décimale de la clé publique
* **E** = l'exposant de la clé publique
* **P** et **Q** = les nombres premiers sachant que P*Q=N
* **D** = e-1 modulo ((p - 1)(q - 1))

(e,n) forme la clé publique.
(d,n) forme la clé privée.

On stockera chaque valeur dans un fichier pour ne pas se perdre.

--- 

 On commence par **N** et **E** en utilisant openssl :
 ```
 openssl rsa -in pubkey.pem -pubin -text -modulus
 ...
 Exponent: 65537 (0x10001)
Modulus=6EA51B0BFF30EC10E94942914B2BA3E7545B4FC20BA4FB1890312476DD068F9
...
 ```
On obtient l'exposant **E**, ainsi que le modolus **N** en hexadécimal.
On le convertit en décimal :
```
python -c "print int('6EA51B0BFF30EC10E94942914B2BA3E7545B4FC20BA4FB1890312476DD068F9', 16)"

31278831112770795370642731531117046729586125732289031514163827433920\61253881
```
 Maintenant qu'on a **N**, on peut calculer **P** et **Q**.
 **N** est seulement composé de 76 digits, il est fort probable que ce modulus est connu.
 Un site permet de trouver ces deux paramètres à partir de **N** -> factordb.com.
On rentre la valeur décimale de **N** et on lance la factorisation : 

---
![](images/factordb.png)

Parfait, on récupère **P** et **Q**.
Il ne nous manque plus que **D**  que l'on calcule avec un script python -> rsatool.py
L'outil nécessite des modules :
* python -> apt install python
* gmyp -> apt install python-gmyp
* pyasn1.codec.der -> apt install python-pyasn1 python-pyasn1-modules

```
python2.7 test.py -p  37489033853046106260847049286846815573 -q 83434615134071502179643767474339282197 -n 3127883111277079537064273153111704672958612573228903151416382743392061253881 -e 65537
Using (p, q) to initialise RSA instance
...
d = 5009bf76fe333591463068a711e98fb33ab20adb0ed26e08b8e2f5d5e24cb1
...
```
On convertit en décimal :
```
python -c "print int('5009bf76fe333591463068a711e98fb33ab20adb0ed26e08b8e2f5d5e24cb1', 16)"

141415042780627533581357726973617665526619654279695600406450825924398173361
```
---
On peut maintenant calculer la clé privée avec rsatool.py :
```
python rsatool.py -f PEM -o private.pem -n 31278831112770795370642731531117046729586125732289031514163827433920\61253881 -d 141415042780627533581357726973617665526619654279695600406450825924398173361
Using (n, d) to initialise RSA instance

...

Saving PEM as private.pem
```

```
cat private.pem

-----BEGIN RSA PRIVATE KEY-----
MIGlAgEAAiAG6lGwv/MOwQ6UlCkUsro+dUW0/CC6T7GJAxJHbdBo+QIDAQABAh9QCb92/jM1kUYw
aKcR6Y+zOrIK2w7Sbgi44vXV4kyxAhAcNCAS5hfelVK4gXsPkJ1VAhA+xOwgV8dg4TBADWjZnX0V
AhAOtxutAKSfm+WHIvekTV3pAhA3p+gu7WYt8r4TrVAw1MHZAhAEYg4kPwPmFFjFOgoavUHf
-----END RSA PRIVATE KEY-----
```

Voilà, on a les données nécessaires pour déchiffrer le message.
Il y a plusieurs façons pour cela :
* Avec un script ( module rsa en python par exemple )
* Avec openssl : openssl rsautl -decrypt -inkey private.key -in encrypted.txt -out plaintext.txt
* Sur des sites

Pour cette fois, on va le faire sur https://8gwifi.org/rsafunctions.jsp.
On rentre les clés ainsi que le texte chiffré, et on récupère le mot de passe du **7Zip** :

---
![](images/7zippass.png)

---
Maintenant que nous avons le mot de passe, nous devons extraire le **7Zip** de l'image.
Pour celà, on peut utiliser binwalk ( apt install binwalk ) :
```
binwalk --dd='zip:zip' logo.png
```
Les fichiers extraits avec binwalk sont placés dans un fichier nommé : _logo.png.extracted.

On décompresse ensuite le fichier **7Zip** que l'on a récupéré, en rentrant le mot de passe après :
```
7za e 2686B9.zip
...
Enter password (will not be echoed):
```
---
On se retrouve avec une capture réseau : echange.pcapng.
On l'ouvre avec wireshark ( apt install wireshark-qt ) :
```
wireshark echange.pcapng
```
On y voit un échange chiffré avec TLS, on a donc besoin d'une clé pour déchiffrer les échanges.
Après avoir cherché dans la capture, on remarque qu'il n'y a rien d'intéressant, il reste peut-être des informations sur l'image.

---
L'outil **stegsolve** va nous débloquer, voici comment l'installer :

```
 wget http://www.caesum.com/handbook/Stegsolve.jar -O stegsolve.jar
```
```
chmod +x stegsolve.jar
```
On l'exécute avec la commande :
```
java -jar stegsolve.jar
```
Cela nécessite d'avoir le **JRE** ( Java Runtime Environment ) :
```
apt install default-jre
```
Stegsolve nous permet de jouer avec les couleurs de l'image.
En manipulant un peu, on arrive sur un QR CODE dans le coin droit :

---
![](images/qrcode.png)

---
On scanne le QR CODE avec une application ( Kaspersky QR Code Scanner dans cet exemple ).
On obtient une clé RSA privée : 
```
-----BEGIN RSA PRIVATE KEY----- 

MIIEpgIBAAKCAQEA7w05qBs1TjeHEJoWtoIMbGThbp1jSTyy3Jc5pNZ3YEGBHQc8
33SkI7cOHr+WOr48uKzUktC6gB9WSusFN2W33lN/++MxeYZVM0Ndfm42GW/Ilvjt
RngANyw2bFkNENpTla2IaHjH7yFFW6Mt0z5/yhVUmCADc9rKgXfZUZy/2RtXu2BF
6zjjopl/27+kJG0WjywYQvZuXbtSRQHktQACa2YwyUnNnoMHuW14a6VcHVtrsZK6
dfdA1x/FEWDEgDRQqzH2rSb52aRiqLwLfuh1SHepb271RmKATXsyjkoUwiPpHAxa
fBt/OfG41fuLGDZdAb3DgooF1LwLyVt/kX/QoQIDAQABAoIBAQDCNkBtbw1zqnE0
77scM7PcLGlp8LxFN+coiEPww7zgmtdGuiaedKkVyxbRrF1lBEFY2DErxiB/DvaQ
jzgBtcYX/kawYfhD/vshVs2Sdf/F0qdo4HwFEGE8cdMVW5Pd6UKXDBESr23fvWwX
8w673L79cy7zGK81ZL+5R6FrSKokuxwDYZcmgiLUHLkBrRac56U2/cJoC5qpt8u5
faKfXAzmWDvsk09L8bGp2N5tqBbeKfVR4C6q4D+oBX95rgD4KnkMjQwAA1Ii5r19
S4kNPjbpWmue1ExOlkK+Mu0sGEpTOqEGvxEo+oNKsbwLsLB8r9BpX+rK/qyZleF5
70SW4FrxAoGBAP4rxAGHUMhn2GK4WoambDIo4XSZil4TrvLCzw6E2QpAYIQQ6oGT
WHgYmV+x1K6AL7nOGdTOXZXxy6BB4zyzGjr4yrnYn2LPajQ52e9E7G8SJfyDTI7d
D8UU9boBijsv+KP6k7SJhKZrjWg9XHb7LU/IGlkjZxdLamtboegVAS0/AoGBAPDF
m1Ty320cBf+MLmXuLJPckxl9KLzkWxVmkxfmWGoHzXKxVkIuWz/RFtS/PuoqVMcB
nwNB8al4qT+ujsTfTFoXYa1Gnk/olf58r+I36TH3sUsZ8iaL7cZpZtFZgfzJl3gg
wl2iX42QYdvyKb3yTMhAsepxtSzsqRlrWxd8USofAoGBALtTLDFoNlyNVIhlkKp+
i21ofcl+w+DUJvNaKlpsBL4udjO/d+Yp32Eb2CYUHUjCxuntAVK7CXRScZ3aEY0R
/zoJVPgltEqRzZdSL9HNfWfa+04gwtLkEBRTf0soiWVX1KueKTA8zhwNVnnGB5jV
RC7xrqGVmwwrE/Kt5ST0HhrnAoGBAMi4RB1WeBKy2QFIcOnC2+DMcsk+APcv/Ark
P14eRrjmFk02L63ujV2vRrRHVwDKF3DsFg8WK4vXrmkLcbQIX9N/SIu2ooYFZP4K
Hmqdp8djMadlUuz1nFzbQ77lOqttt9uKm96JXdVbd2lwAYJ7e6052diTwLFZ/kLV
r9PtbJCLAoGBAIw9hqLhViIUuxW6EEyGsSUYxViny8xwYiMBcWZ17gwehwk3R35b
cNSRglyp1pmaxrYT8lSonI84BRlpaPk/bhyI83PLmoFZFMTM9rNSVgiTPYKDE/Zu
DlAeV7QzbQ59tb0YjoZ2DbnI3aP3oFtjbgej5/8hBSbzeLJ66fYcdC8k

-----END RSA PRIVATE KEY-----
```
Cette clé est sans doute celle du certificat.

---
Pour ajouter la clé dans wireshark :
Edit > Preferences > Protocols > SSL
RSA keys list > Edit
* IP address -> 192.168.0.87
* Port -> 443
* Protocol -> http 
* Key File -> La clé qu'on a récupéré, dans un fichier 'private.key' par exemple
* Password -> Aucun, la clé n'est pas chiffrée

OK

---

Deux champs sont devenus verts, on remarque qu'un zip est téléchargé. On va donc l'extraire de la capture :
File > Export Objects > HTTP
capture.zip > Save

On essaye de le décompresser mais il est protégé par un mot de passe.
La commande ```unzip -l capture.zip``` nous permet de lister le contenu de l'archive.
Elle nous révèle aussi les commentaires, où il est marqué : 
* un fichier GAIO.BIN > COMPRESSED WITH PKZIP 1.10 OLD BUT GOOD SOFTWARE !
* un fichier CERTIF.PNG > CERTIFICAT AUTHENTICITE GAIO

Les **PKZIP 1.10** ont une vulnérabilité de type 'attaque par clair connu'.
Si on possède une partie de l'archive en clair, on peut comparer le fichier chiffré avec le fichier en clair.

---
Le commentaire ```CERTIFICAT AUTHENTICITE GAIO```  nous donne un indice.
On avait vu ce certificat dans l'image, lors de l'extraction des données ```(zsteg -a logo.png)``` :
```
"GAIO 2016 - BONE / MAURUGEON - Certificat d'authenticité : 
89504e470d0a1a0a0000000d4948445200000167000001460802000000392ac949000020004944415478daec7d77985
...
27dff204f1cf46f69b1f35fce637bffd4afb7fc46923a7fc7894730000000049454e44ae426082"
```

Le fichier de l'archive étant un fichier au format PNG, on va convertir le certificat que l'on a au format PNG. On stocke la valeur dans un fichier puis on passe la commande :
```
xxd -r -p certificat > certificat.png
```
**-r** > Hex to Bin
**-p** > Sortie en hexadécimal

Pour comparer les deux fichiers, on utilise l'outil **pkcrack** :
```
wget -c https://www.unix-ag.uni-kl.de/~conrad/krypto/pkcrack/pkcrack-1.2.2.tar.gz
```
```
gunzip pkcrack-1.2.2.tar.gz && tar xvf pkcrack-1.2.2.tar
```
```
cd pkcrack-1.2.2/src/ && make
```
On extrait d'abord le fichier chiffré pour pouvoir le comparer au fichier en clair :
```
./extract capture.zip CERTIF.PNG
```
Le fichier chiffré est plus long de 12 bytes que le fichier en clair, l'extraction est réussie.

On peut enfin lancer l'attaque par clair connu :
```
./pkcrack -c CERTIF.PNG -p certificat.png 
```
L'outil trouve le mot de passe D3pT.CyB3r, on peut décompresser l'archive.

---

On se retrouve maintenant avec un exécutable ( ELF 32bits LSB ).
On l'éxecute avec le paramètre demandé, une erreur apparaît :

On décide de le lancer avec gdb ( GNU Project Debugger ) :
```gdb ./GAIO.BIN```
On remarque que seul la première lettre influence le crash du programme. On décide alors de bruteforce le premier caractère.
On se rend compte que le programme crash 'le plus loin' avec les lettres **H** et **T**.
On obtient potentiellement le premier caractère du mot de passe.

---
On décompile ensuite le programme dans IDA pour analyser les fonctions du programme.

Tout d'abord, on voit que le programme charge une adresse dans le registre, il affichera à cette adresse le message du challenge.

---
![](images/ida.png)

---
![](images/ida2.png)

---
Il charge à nouveau une adresse dans le même registre, mais cette fois la donnée n'est pas lisible.

---
![](images/ida4.png)

---
Il vérifie que l'un des paramètres auquel le programme est lancé avec soit bien égal à 2.

---
![](images/ida3.png)

---
Et on remarque la partie la plus intéressante, la boucle.

---
![](images/ida5.png)

---
Explication de la boucle :
* `mov     eax, [ebp+0Ch]`
Déplace dans eax l'adresse de début de la chaîne dans charactères qu'on donne au programme (le mot de passe).
* `movx   eax, byte ptr [eax+edi]`
Récupère 8 bits à l'adresse eax + edi -> On récupère la lettre du mot de passe à l'index edi.
`char eax = password[edi]`
* `xor [ecx+ebx], eax`
On xor ce qu'il y a à l'adresse contenu dans ecx à l'index ebx avec la lettre du mot de passe qu'on a chargé pour l'instant (ici la première).
Ce qu'il y a à l'adresse dans  ecx à ce moment est la donnée illisible qu'on a vu tout à l'heure.
* `inc ebx`
On ajoute 1 à ebx.
* `cmp ebx, edx`
On compare les deux valeurs ebx et edx.
* `jnz short loc_80480AA`
En fonction de la comparaison au dessus, on revient au début de loc_8048011 si les valeurs sont différentes, sinon on continue.
* `jmp ecx`
On continue l'execution à l'adresse contenue dans ecx.
---
On voit qu'on a une boucle qui effectue une action "edx" fois puis sort.
On charge 0x11 (17) dans edx, donc cette boucle, boucle 17 fois (on peut le vérifier avec gdb et des breakpoints).

On sait que H ou T pourraient être la première lettre, et on sait aussi que le programme XOR la lettre avec la donnée plus haut. On récupère les 17 bytes qu'on met dans un script python qui va servir à nous faire le XOR.

---
![](images/script.png)

---
Ce script permet d'effectuer un XOR avec la valeur de la lettre qu'on lui attribue et il nous retourne les valeurs (bytes).

On copie le code retourné sur ce site https://onlinedisassembler.com/odaweb/ il nous retourne ça, après avoir réglé les paramètres (architecture, adresse du début) :

---
![](images/online.png)

---
* `xor ebx, ebx`
Remet le registre ebx à 0
* `mov ecx,0x8049166`
On remet dans ecx l'adresse des données qui sont illisibles. On suppose donc que comme pour ce qu'on fait maintenant, c'est illisible et que ça va être XOR avec la seconde lettre du mot de passe.
 * `mov dl, 0x11`
On remet 17 dans edx.
* `mov esi, 0x80480aa`
On charge l'adresse deu début de la boucle dans esi.
* `inc edi`
On incrémente edi (qui était le compteur pour quelle lettre du mot de passe est utilisée).
* `jmp esi`
On jump sur l'adresse dans esi, qui est le début de la boucle.
---
A ce stade esi est incrémenté, on utilise maintenant le second caractère du mot de passe. On ne peut donc pas bruteforce à la main.

Comme on doit toujours jump sur l'adresse 0x80480aa, on peut se dire que la valeur "aa" sera présente dans les bytes après avoir été XOR. On peut donc faire un petit ajout à notre script python pour vérifier ça.

---
![](images/script2.png)

---
Resultat :
```
0 has 0xaa
A has 0xaa
e has 0xaa
```
On se retrouve donc avec seulement 3 possibilités pour ce second caractère. Après avoir retester avec le script, on trouve que 0 est le second caractère.

On retourne sur le site, on redonne les bytes qui ont été XOR et on peut ainsi voir que le code est le même qu'avant.

---
On recommence le processus pour chaque caractère jusqu'à trouver le mot de passe.

Le script final :

---
![](images/script3.png)

---
On met le mot de passe dans une variable et on exécute le binaire :
```
a=H0pp3R

./GAIO.BIN $a
Challenge GAIO/Cyber 2
Vous etes sur la bonne voie !
VicToire !
```

Flag : H0pp3R

---

Challenge réalisé avec :

* <a href="https://github.com/mohamedlazar" title="link to mohamed git">Mohamed Lazar</a>

* Théo Hay
