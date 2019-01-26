# YubiKey Doku

Diese Doku erklärt wie man den Yubikey einrichtet und zur Verbindung per SSH verwenden kann.

## Inhaltsverzeichnis
- [YubiKey Doku](#yubikey-doku)
  - [Inhaltsverzeichnis](#inhaltsverzeichnis)
  - [Installation](#installation)
    - [Einrichten der Umgebung](#einrichten-der-umgebung)
    - [Generieren der GPG-Keys](#generieren-der-gpg-keys)
    - [Sichern der GPG-Keys](#sichern-der-gpg-keys)
    - [Konfigurieren des Yubikey](#konfigurieren-des-yubikey)
    - [Aufspielen des Sub-Keys auf den Yubikey](#aufspielen-des-sub-keys-auf-den-yubikey)
    - [Exportieren des Public-Keys](#exportieren-des-public-keys)
  - [Client](#client)
    - [Einrichten der Keys für SSH auf MacOS](#einrichten-der-keys-f%C3%BCr-ssh-auf-macos)
    - [Wechseln des Yubikeys](#wechseln-des-yubikeys)
  - [Reset des Yubikey](#reset-des-yubikey)


## Installation
### Einrichten der Umgebung
Ich habe eine VM-Ware Maschiene verwendt, um die GPG-Keys zu generieren. Natürlich könnte man wenn man noch mehr auf Sicherheit bedacht ist, auch eine komplett Air-Gapped Maschiene benutzen.

Ich habe ein Standard Ubuntu benutzt, andere UNIX/Linux basierte Systeme sollten aber ähnlich funktionieren.

Zuerst muss die VM-Ware Maschiene so eingerichtet werden, dass zu dieser auch der Yubikey durchgemapt werden kann, dafür muss in dem .vmx file der Maschiene, folgendes eingefügt werden:
```
usb.generic.allowHID = "TRUE"
usb.generic.allowLastHID = "TRUE"
```
> Wichtig ist, dass die VM nicht verschlüsselt ist, falls doch, muss diese zuvor entschlüsselt werden

Nun kann man den Key ganz nochmal über VM-Ware verbinden.

Nun können die Dependencies installiert werden
``` bash
sudo apt-get install haveged gnupg2 gnupg-agent libpth20 pinentry-curses libccid pcscd scdaemon libksba8 paperkey opensc
```

Um mit dem Yubikey zu arbeiten wird gnupg2 >= 2.0.22 und scdaemon >= 2.0.22 benötigt

### Generieren der GPG-Keys
Ich habe einen Master-Key erstellt, und Sub-Key für Signierung, Authentifizierung (SSH) und Verschlüsselung.
```
pub  4096R/0x2896DB4A0E427716  created: 2015-04-15  expires: 2016-04-14  usage: SC
                               trust: ultimate      validity: ultimate
sub  2048R/0x770A210849C9CBD7  created: 2015-04-15  expires: 2015-10-12  usage: S
sub  2048R/0x6BF07F7DA7D84FFD  created: 2015-04-15  expires: 2015-10-12  usage: E
sub  2048R/0x7B13B2E1879F1ED3  created: 2015-04-15  expires: 2015-10-12  usage: A
[ultimate] (1). Test User <test@test.com>
[ultimate] (2)  Test User <test@megatestcorp.ca>
[ultimate] (3)  Test User <test@keybase.io>
```

Am Anfang starten wir mit der Erstellung eines Master-Keys. hier kann, auch wenn Yubikey nur 2048-bit Keys unterstützt durchaus ein 4096-bit Key verwendet werden. Auch ein Ablaufdatum sollte gesetzt werden, sodass eventuell gestohlene Keys nicht auf irgendwelchen Key-Servern noch ewig rumliegen.

```
gpg --full-generate-key
gpg (GnuPG) 2.2.12; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
Your selection? 4
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 4y
Key expires at Tue 24 Jan 2023 03:40:14 PM EST
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

Real name: Yannick Stein
Email address: y@nnick.me
Comment: YubiKey-2
You selected this USER-ID:
    "Yannick Stein (YubiKey-2) <y@nnick.me>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
gpg: key 0xC7C50C3E030B3220 marked as ultimately trusted
gpg: revocation certificate stored as '/root/.gnupg/openpgp-revocs.d/76A4A8AABCCA8C707BFF36B9C7C50C3E030B3220.rev'
public and secret key created and signed.

Note that this key cannot be used for encryption.  You may want to use
the command "--edit-key" to generate a subkey for this purpose.
pub   rsa4096/0xC7C50C3E030B3220 2019-01-25 [SC] [expires: 2023-01-24]
      Key fingerprint = 76A4 A8AA BCCA 8C70 7BFF  36B9 C7C5 0C3E 030B 3220
uid                              Yannick Stein (YubiKey-2) <y@nnick.me>
```

Nun können wir die Sub-Keys erstellen. Dies müssen beim Yubijey dann 2048-bit Keys sein.

```
gpg2 --expert --edit-key 0xC7C50C3E030B3220
gpg (GnuPG) 2.2.12; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

gpg: checking the trustdb
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: depth: 0  valid:   2  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 2u
gpg: next trustdb check due at 2021-01-24
sec  rsa4096/0xC7C50C3E030B3220
     created: 2019-01-25  expires: 2023-01-24  usage: SC  
     trust: ultimate      validity: ultimate
[ultimate] (1). Yannick Stein (YubiKey-2) <y@nnick.me>

gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 4
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 2048
Requested keysize is 2048 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 4y
Key expires at Tue 24 Jan 2023 03:43:28 PM EST
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa4096/0xC7C50C3E030B3220
     created: 2019-01-25  expires: 2023-01-24  usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/0xD7697A298E606058
     created: 2019-01-25  expires: 2023-01-24  usage: S   
[ultimate] (1). Yannick Stein (YubiKey-2) <y@nnick.me>

gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 6
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 2048
Requested keysize is 2048 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 4y
Key expires at Tue 24 Jan 2023 03:44:11 PM EST
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa4096/0xC7C50C3E030B3220
     created: 2019-01-25  expires: 2023-01-24  usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/0xD7697A298E606058
     created: 2019-01-25  expires: 2023-01-24  usage: S   
ssb  rsa2048/0xDAE97EDCE6331918
     created: 2019-01-25  expires: 2023-01-24  usage: E   
[ultimate] (1). Yannick Stein (YubiKey-2) <y@nnick.me>
```
Nun müssen wir noch den Authenticate Key erstellen. Da dies nicht zur Option steht, müssen wir einen RSA-Key mit eingenen Eigenschaften erstellen. Hier ist die Standardeinstellung `Sign` und `Encrypt`. Diese Einstellungen wählen wir beide ab, und wählen stattdessen `Authenticate`.
```
gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 8

Possible actions for a RSA key: Sign Encrypt Authenticate 
Current allowed actions: Sign Encrypt 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? s

Possible actions for a RSA key: Sign Encrypt Authenticate 
Current allowed actions: Encrypt 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? e

Possible actions for a RSA key: Sign Encrypt Authenticate 
Current allowed actions: 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? a

Possible actions for a RSA key: Sign Encrypt Authenticate 
Current allowed actions: Authenticate 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? q
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 2048
Requested keysize is 2048 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 4y
Key expires at Tue 24 Jan 2023 03:45:34 PM EST
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa4096/0xC7C50C3E030B3220
     created: 2019-01-25  expires: 2023-01-24  usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/0xD7697A298E606058
     created: 2019-01-25  expires: 2023-01-24  usage: S   
ssb  rsa2048/0xDAE97EDCE6331918
     created: 2019-01-25  expires: 2023-01-24  usage: E   
ssb  rsa2048/0x9A4801FEF3AD0869
     created: 2019-01-25  expires: 2023-01-24  usage: A   
[ultimate] (1). Yannick Stein (YubiKey-2) <y@nnick.me>

gpg> save
```
Nun haben wir die Key erstellt, und können mit dem Sichern fortfaheren.

### Sichern der GPG-Keys
Nun können wir die Keys auf einen USB-Stick oder ähnliches sichern. Dafür können wir einmal den Ordner `~/.gnupg` auf den USB-Stick sichern, und auch die Keys direkt exportieren.
```
tar -czf /media/BACKUP/gnupg.tgz ~/.gnupg
gpg2 -a --export-secret-key  0xC7C50C3E030B3220 >> /media/BACKUP/C7C50C3E030B3220.master.key
gpg2 -a --export-secret-subkeys  0xC7C50C3E030B3220 >> /media/BACKUP/C7C50C3E030B3220.sub.key
```
> Nicht vergessen auch die Passphrase zu sichern, sodass man diese Datensicherung auch benutzen kann.

### Konfigurieren des Yubikey
Den Yubikey kann man mit `gpg2 -card-edit` konfigurieren. Hier konfigurieren wir zuerst die User und Admin PINs. Außerdem können wir eine URL angeben, wo der Private-Key gedownloadet werden kann.

Standardmäßig ist der User PIN vom Yubikey auf `123456` und der Admin PIN auf `12345678`

```
gpg2 --card-edit

Reader ...........: 1050:0116:X:0
Application ID ...: D2760001240102000006065323360000
Version ..........: 2.0
Manufacturer .....: Yubico
Serial number ....: 06532336
Name of cardholder: [not set]
Language prefs ...: [not set]
Sex ..............: unspecified
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]

gpg/card> admin
Admin commands are allowed

gpg/card> passwd
gpg: OpenPGP card no. D2760001240102000006065323360000 detected

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 1
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 3
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? q

gpg/card> name
Cardholder's surname: Yannick
Cardholder's given name: Stein

gpg/card> sex
Sex ((M)ale, (F)emale or space): m

gpg/card> land

Invalid command  (try "help")

gpg/card> lang
Language preferences: de

gpg/card> list

Reader ...........: 1050:0116:X:0
Application ID ...: D2760001240102000006065323360000
Version ..........: 2.0
Manufacturer .....: Yubico
Serial number ....: 06532336
Name of cardholder: Stein Yannick
Language prefs ...: de
Sex ..............: male
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]

gpg/card> quit

```

### Aufspielen des Sub-Keys auf den Yubikey
Nun spielen wir die Sub-Keys auf die Yubikeys. Hierfür müssen immer die jeweiligen Keys markiert und nachher transferiert werden.

```
gpg2 --edit-key 0xC7C50C3E030B3220
gpg (GnuPG) 2.2.12; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

sec  rsa4096/0xC7C50C3E030B3220
     created: 2019-01-25  expires: 2023-01-24  usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/0xD7697A298E606058
     created: 2019-01-25  expires: 2023-01-24  usage: S   
ssb  rsa2048/0xDAE97EDCE6331918
     created: 2019-01-25  expires: 2023-01-24  usage: E   
ssb  rsa2048/0x9A4801FEF3AD0869
     created: 2019-01-25  expires: 2023-01-24  usage: A   
[ultimate] (1). Yannick Stein (YubiKey-2) <y@nnick.me>

gpg> toggle

sec  rsa4096/0xC7C50C3E030B3220
     created: 2019-01-25  expires: 2023-01-24  usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/0xD7697A298E606058
     created: 2019-01-25  expires: 2023-01-24  usage: S   
ssb  rsa2048/0xDAE97EDCE6331918
     created: 2019-01-25  expires: 2023-01-24  usage: E   
ssb  rsa2048/0x9A4801FEF3AD0869
     created: 2019-01-25  expires: 2023-01-24  usage: A   
[ultimate] (1). Yannick Stein (YubiKey-2) <y@nnick.me>

gpg> key 1

sec  rsa4096/0xC7C50C3E030B3220
     created: 2019-01-25  expires: 2023-01-24  usage: SC  
     trust: ultimate      validity: ultimate
ssb* rsa2048/0xD7697A298E606058
     created: 2019-01-25  expires: 2023-01-24  usage: S   
ssb  rsa2048/0xDAE97EDCE6331918
     created: 2019-01-25  expires: 2023-01-24  usage: E   
ssb  rsa2048/0x9A4801FEF3AD0869
     created: 2019-01-25  expires: 2023-01-24  usage: A   
[ultimate] (1). Yannick Stein (YubiKey-2) <y@nnick.me>

gpg> keytocard
Please select where to store the key:
   (1) Signature key
   (3) Authentication key
Your selection? 1

sec  rsa4096/0xC7C50C3E030B3220
     created: 2019-01-25  expires: 2023-01-24  usage: SC  
     trust: ultimate      validity: ultimate
ssb* rsa2048/0xD7697A298E606058
     created: 2019-01-25  expires: 2023-01-24  usage: S   
ssb  rsa2048/0xDAE97EDCE6331918
     created: 2019-01-25  expires: 2023-01-24  usage: E   
ssb  rsa2048/0x9A4801FEF3AD0869
     created: 2019-01-25  expires: 2023-01-24  usage: A   
[ultimate] (1). Yannick Stein (YubiKey-2) <y@nnick.me>

gpg> key 1

sec  rsa4096/0xC7C50C3E030B3220
     created: 2019-01-25  expires: 2023-01-24  usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/0xD7697A298E606058
     created: 2019-01-25  expires: 2023-01-24  usage: S   
ssb  rsa2048/0xDAE97EDCE6331918
     created: 2019-01-25  expires: 2023-01-24  usage: E   
ssb  rsa2048/0x9A4801FEF3AD0869
     created: 2019-01-25  expires: 2023-01-24  usage: A   
[ultimate] (1). Yannick Stein (YubiKey-2) <y@nnick.me>

gpg> key 2

sec  rsa4096/0xC7C50C3E030B3220
     created: 2019-01-25  expires: 2023-01-24  usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/0xD7697A298E606058
     created: 2019-01-25  expires: 2023-01-24  usage: S   
ssb* rsa2048/0xDAE97EDCE6331918
     created: 2019-01-25  expires: 2023-01-24  usage: E   
ssb  rsa2048/0x9A4801FEF3AD0869
     created: 2019-01-25  expires: 2023-01-24  usage: A   
[ultimate] (1). Yannick Stein (YubiKey-2) <y@nnick.me>

gpg> keytocard
Please select where to store the key:
   (2) Encryption key
Your selection? 2

sec  rsa4096/0xC7C50C3E030B3220
     created: 2019-01-25  expires: 2023-01-24  usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/0xD7697A298E606058
     created: 2019-01-25  expires: 2023-01-24  usage: S   
ssb* rsa2048/0xDAE97EDCE6331918
     created: 2019-01-25  expires: 2023-01-24  usage: E   
ssb  rsa2048/0x9A4801FEF3AD0869
     created: 2019-01-25  expires: 2023-01-24  usage: A   
[ultimate] (1). Yannick Stein (YubiKey-2) <y@nnick.me>

gpg> key 2

sec  rsa4096/0xC7C50C3E030B3220
     created: 2019-01-25  expires: 2023-01-24  usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/0xD7697A298E606058
     created: 2019-01-25  expires: 2023-01-24  usage: S   
ssb  rsa2048/0xDAE97EDCE6331918
     created: 2019-01-25  expires: 2023-01-24  usage: E   
ssb  rsa2048/0x9A4801FEF3AD0869
     created: 2019-01-25  expires: 2023-01-24  usage: A   
[ultimate] (1). Yannick Stein (YubiKey-2) <y@nnick.me>

gpg> key 3

sec  rsa4096/0xC7C50C3E030B3220
     created: 2019-01-25  expires: 2023-01-24  usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/0xD7697A298E606058
     created: 2019-01-25  expires: 2023-01-24  usage: S   
ssb  rsa2048/0xDAE97EDCE6331918
     created: 2019-01-25  expires: 2023-01-24  usage: E   
ssb* rsa2048/0x9A4801FEF3AD0869
     created: 2019-01-25  expires: 2023-01-24  usage: A   
[ultimate] (1). Yannick Stein (YubiKey-2) <y@nnick.me>

gpg> keytocard
Please select where to store the key:
   (3) Authentication key
Your selection? 3

sec  rsa4096/0xC7C50C3E030B3220
     created: 2019-01-25  expires: 2023-01-24  usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/0xD7697A298E606058
     created: 2019-01-25  expires: 2023-01-24  usage: S   
ssb  rsa2048/0xDAE97EDCE6331918
     created: 2019-01-25  expires: 2023-01-24  usage: E   
ssb* rsa2048/0x9A4801FEF3AD0869
     created: 2019-01-25  expires: 2023-01-24  usage: A   
[ultimate] (1). Yannick Stein (YubiKey-2) <y@nnick.me>

gpg> save
```

### Exportieren des Public-Keys
Der Public-Key kann mit folgendem Befehl in die Datei *key.asc* exportiert werden
```
gpg2 -a --export 0xC7C50C3E030B3220 >> key.asc
```

## Client
### Einrichten der Keys für SSH auf MacOS
Die Installation von gpg kann mit `brew install gnupg2` durchgeführt werden.


Nun kann der Key importiert werden. Dies kann anz einfach über folgenden Befehl durchgeführt werden:
```
gpg --import key.asc
```
*key.asc* sollte natürlich durch den Public-Key ergänzt werden.
Der Key sollte nun bei gestecktem Yubikey mit `gpg -K` aufgelistet werden.


Danach müssen folgende Einstellungen vorgenommen werden, damit SSH die GPG-Keys verwendet

Hinzufügen oder Editieren der Datei `~/.gnupg/gpg-agent.conf`:
```
default-cache-ttl 600
max-cache-ttl 7200
enable-ssh-support
```

Hinzufügen oder Editieren der Datei `~/.bash_profile`:
``` bash
# Added for SSH - GPG Support
gpgconf --launch gpg-agent
GPG_TTY=$(/usr/bin/tty)
SSH_AUTH_SOCK="$HOME/.gnupg/S.gpg-agent.ssh"
export GPG_TTY SSH_AUTH_SOCK
```

Anschließend ist ein Neustart des Macs notwendig.

Der Public-SSH-Key kann nun mit `ssh-add -L` exportiert werden, um ihn auf den Servern in den uthorized_keys einzutragen.

### Wechseln des Yubikeys
Falls man einen anderen Yubikey mit dem selben Schlüssel verwenden will (beispielsweise weil man den Schlüssel aus dem Backup wiederhergestellt hat), dann muss man auf den Clienten den Verweis auf den alten Yubikey löschen. 

Dies geht mit
```
rm ~/.gnupg/private-keys-v1.d/*
```
Anschließend muss der neue eingesteckt Yubikey mit `gpg --card-status`neu eingelesen werden.

Nun kann der neue Yubikey so wie der alte verwendet werden.


## Reset des Yubikey
Der Yubikey kann mit dem CLI von Yubico auch zurückgesetzt werden.
```
ykman openpgp reset
WARNING! This will delete all stored OpenPGP keys and data and restore factory settings? [y/N]: y
Resetting OpenPGP data, don't remove your YubiKey...
Success! All data has been cleared and default PINs are set.
PIN:         123456
Reset code:  NOT SET
Admin PIN:   12345678
```

Vielen Dank an:
+ [Superuser Forum (EN)](https://superuser.com/questions/1284632/unable-to-get-yubikey-neo-u2f-working-in-linux-inside-of-vmware-workstation)
+ [FROGESLNET (EN)](https://www.forgesi.net/gpg-smartcard/)
