+++
title = "Offline & Encrypted Personal E-Mail"
description = "Managing e-mail with mutt, isync, msmtp, tar, and gpg2"
date = 2021-02-16
+++

&nbsp;

I find most e-mail clients to be both complicated and restrictive.
I like being able to read, write and manage my e-mail offline.
I also like storing my e-mail in an encrypted mailbox, in case someone else gets access to my device.

In this guide, I will explain how I have configured my e-mail system, and how it can be implemented for other users.
I will be using an exemplary Yandex e-mail account (example@yandex.com).

The programs needed to replicate this setup are:

+ `mutt` (e-mail client)
+ `isync` (IMAP synchronizer)
+ `msmtp` (SMTP client)
+ `gnupg2` (encryption and signing tool)
+ `pass` (password manager)

These programs are easily available in most Linux and BSD systems, even though their package names might be different.
For this guide, Void Linux is used.
Instructions for other Linux or BSD systems should be very similar.

---

Before we begin, we need to install the required packages.

```sh
user@void: ~$ doas xbps-install mutt isync msmtp gnupg2 pass
```

We start by configuring `mutt`, which is a very flexible and customizable e-mail client with many options.
The lines below are the ones that are relevant to this guide.
If you are already a `mutt` user, then these lines should be enough.
If not, then you might need to tweak the configuration to make it more usable for you.

We open the configuration file in a text editor.

```sh
user@void: ~$ vim ~/.config/mutt/muttrc
```

We add the following lines, which tell `mutt` how to send e-mail, and where to look for received e-mail.

```sh
set folder = "$HOME/.email" # All e-mail
set spoolfile = "$HOME/.email/INBOX" # "Inbox" folder
set record = "$HOME/.email/Sent" # "Sent" folder
set sendmail = "/usr/bin/msmtp -a yandex" # Use msmtp
set use_from = yes # Generate the "From:" header field
set from = example@yandex.com # E-mail address header field
set realname = "Example User" # Real name
set mbox = "$HOME/.mailbox/Archives" # Read e-mail
set mbox_type = Maildir # Preferred mailbox type/format
set move # Automatically move read e-mail to "mbox"
```

Next, we configure `isync` to get e-mail from a remote IMAP server.
The binary for `isync` is called `mbsync`, and the default configuration file is `~/.mbsyncrc`.
In this guide, we will put the configuration file in `~/.config/mbsync/mbsyncrc`, and read the file when running `mbsync`.
This makes our home directory tidier.

```sh
user@void: ~$ mkdir -p ~/.config/mbsync
user@void: ~$ vim ~/.config/mbsync/mbsyncrc
```

A basic configuration looks something like this.

```sh
IMAPStore yandex-remote # Remote IMAP server name
Host imap.yandex.com # Remote IMAP server address
Port 993 # IMAP port
User example@yandex.com # User e-mail account
PassCmd "pass yandex" # Command to get account password
SSLType IMAPS # Connection security/encryption method
CertificateFile /etc/ssl/certs/ca-certificates.crt

MaildirStore yandex-local # Local e-mail folder name
Path ~/.mailbox/ # Path to local e-mail folder
Inbox ~/.mailbox/INBOX # Path to local inbox
Subfolders Verbatim # Local folder naming style

Channel yandex # Synchronization channel
Master :yandex-remote: # Remote server
Slave :yandex-local # Local storage
Create Both # Create missing mailboxes on both
Expunge Both # Remove all e-mail marked for deletion
Patterns * # Synchronize all mailboxes
SyncState * # Hidden synchronization state file in mailbox
```

Then, we open up the `msmtp` configuration file, which is read from `~/.config/msmtp/config` by default.

```sh
user@void: ~$ vim ~/.config/msmtp/config
```

Here is a simple configuration that works.

```sh
defaults # Set defaults for all accounts
auth on # Enable authentication
tls on # Enable TLS for secure connections
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile /tmp/msmtp.log # Enable logging (optional)

account yandex # Start a new account
host smtp.yandex.com # SMTP server
port 465 # SMTP port
from example@yandex.com # Set the "From:" address
user example@yandex.com # User account
passwordeval "pass yandex" # Command to get account password
tls_starttls off # Tunnel session through TLS

account default : yandex # Default account
```
Next, we generate two RSA keypairs with `gnupg2` (one for the mailbox encryption, and one for the encrypted password storage).
The binary is called `gpg2` on Void Linux.

```sh
user@void: ~$ gpg2 --full-generate-key
```

The program will ask us questions about the keys we are generating.
We need to:

+ Choose "RSA and RSA" keys (this is default)
+ Choose 4096 bits as the keysize for better security (type 4096 when prompted for the keysize)
+ Specify a duration of validity for the keys (this is optional, but is a good security measure, and the key can always be extended when necessary)
+ Confirm the choices
+ Enter a real name (Example User)
+ Enter an e-mail address (example@yandex.com)
+ Enter a comment (this is optional)
+ Press "O" for "Okay"
+ Enter the passphrase to protect the new keys
+ Perform random actions on the mouse and on the keyboard in order to gain entropy while generating the keys (this is optional, and fun).

Then, we enter the following command to see the secret key ID.

```sh
user@void: ~$ gpg2 --list-keys --keyid-format LONG
```

The line beginning with `pub` should contain the secret key ID (`rsa4096/<key_id>`). We note this number.

```txt
/home/example/.gnupg/pubring.kbx
--------------------------------
pub   rsa4096/0000000000000000 2021-01-01 [SC] # The 16-digit number here is the secret key ID.
      0000000000000000000000000000000000000000
uid   [ultimate] Example User <example@yandex.com>
sub   rsa4096/0000000000000000 2021-01-01 [E]
```

We then generate a new RSA keypair with the same process.
This time, we might provide a comment when prompted, in order to make the new keypair more recognizable.
When the generation is complete, we note the ID of the new key as well.

Next up, we initialize the password storage with `pass`, using one of the key IDs we have generated.

```sh
user@void: ~$ pass init "0000000000000000"
```
By default, `pass` saves passwords in a `gpg2` encrypted file in the `~/.password-store` directory.
We are going to store our e-mail account password with `pass`, because it is more secure than storing it in plain text.
We can insert a new password with the following command (we enter the password when prompted).

```sh
user@void: ~$ pass insert yandex
```

We can see the newly added password like this:

```sh
user@void: ~$ pass ls
```

We also need to make the mailbox directory, compress it into a `tar` archive, and encrypt it using the other RSA keypair.
We will be decrypting the mailbox whenever we synchronize with `mbsync` or run the `mutt` e-mail client, and encrypting it again when we are done.
I am using a different fake key ID this time, to differentiate it from the previous one.

We run the following commands first to make the encrypted `tar` archive containing the empty mailbox, and remove the unencrypted files.

```sh
user@void: ~$ mkdir ~/.mailbox
user@void: ~$ tar -cf ~/.mailbox.tar -C ~/ .mailbox
user@void: ~$ rmdir ~/.mailbox
user@void: ~$ gpg2 -r 1111111111111111 -e ~/.mailbox
user@void: ~$ rm ~/.mailbox.tar
```

Finally, we write a simple shell script that will run everything as required. Let's call the script `mymail`.

```sh
user@void: ~$ vim mymail
```
Our script will:

+ Decrypt the encrypted `tar` archive, and make a backup of the encrypted one, in case something goes wrong.
+ Extract the mailbox from the archive, and remove the archive.
+ Run `mbsync` to get the latest e-mail from the remote server.
+ Run `mutt`
+ When `mutt` is closed, compress the mailbox into a `tar` archive, and remove the uncompressed mailbox.
+ Encrypt the archived mailbox, and remove the unencrypted one.

Here is the script:

```sh
#!/bin/sh
MBOX="~/.mailbox"
gpg2 -o $MBOX.tar -d $MBOX.tar.gpg && \
mv $MBOX.tar.gpg $MBOX.tar.gpg.bak
tar -xf $MBOX.tar -C ~/ && rm $MBOX.tar
[ -d "$MBOX" ] && mbsync -c ~/.config/mbsync/mbsyncrc -a
mutt
tar -cf $MBOX.tar -C ~/ .mailbox && rm -r $MBOX
gpg2 -r example@yandex.com -e $MBOX.tar && \
rm $MBOX.tar
```

We save the script and make it executable.

```sh
user@void: ~$ chmod +x mymail
```

Then, we copy the script to a directory in our `$PATH`, or we just run it.

```sh
user@void: ~$ ./mymail
```
