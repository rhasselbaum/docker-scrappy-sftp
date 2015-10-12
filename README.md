# docker-scrappy-sftp
Scrappy SFTP is a multi-user SFTP server that emails transfer logs. Each SFTP user has read-write access to their own private directory in a chroot jail. A special `sftpadmin` user has access to all users' directories and can upload files to any of them, which can then be downloaded (or removed) by the individual users.

The implementation is based on a [blog post](https://publicstringblog.wordpress.com/) I wrote a while back about setting up a chrooted SFTP server on Ubuntu, but I've added a few improvements since then and wrapped it up in a nice, portable Docker image.

# Quick tour
If you just want to try it out, start the container like this:

```
docker run -it --rm --name sftp -p 2022:22 rhasselbaum/scrappy-sftp
```

The container runs an [OpenSSH server](http://www.openssh.com/) and a file transfer mailer daemon. The SSH/SFTP server is exposed as port 2022 on host. Naturally, you can change this to whatever you like including the SSH default port 22 if it isn't already in use on the host.

You can't do much with the server, though, until you create a user. In a separate terminal, run the following:

```
docker exec -it sftp addsftpuser testuser
```

Follow the prompts to set a password. You can leave the other attributes blank if you want. Once it's done, you can try uploading a file. Make sure you have the `sftp` client installed on your host. Then, in the same terminal, run:

```
echo "Hello world" >foo.txt
sftp -P 2022 testuser@localhost
```

And at the SFTP prompt:

```
sftp> put foo.txt
sftp> quit
```

Congratulations, you uploaded your first file! If the container had been configured to email transfer logs, an email would have been sent out at this point. We'll set that up later. For now, let's connect as the `sftpadmin` user to retrieve the uploaded file. Run:

```
sftp -P 2022 sftpadmin@localhost
```

The default password for the `sftpadmin` user is `sftpadmin`. When logged in as this user, we can browse all users' directories, as well as upload, download, and change files and subdirectories.

```
sftp> ls
dev       testuser  
sftp> cd testuser/data
sftp> ls
foo.txt  
sftp> get foo.txt
Fetching /testuser/data/foo.txt to foo.txt
/testuser/data/foo.txt                                                                                        100%   12     0.0KB/s   00:00    
sftp> rm foo.txt
Removing /testuser/data/foo.txt
sftp> quit
```

Neat! But we should really change the `sftpadmin` user's password, which we can do with:

```
docker exec -it sftp passwd sftpadmin
```

The same command can be used to change any other user's password. All of the users are regular local Linux users, so they can be managed through the standard utilities executed via `docker exec`. Alternatively, you can use public key authentication by placing a `.ssh/authorized_keys` file with appropriate permissions in a user's home directory. When you're done experimenting, you can hit CTRL-C in the terminal that started the Scrappy SFTP container to shut it down and clear out the test data.

If you just want a small server and you're not concerned about backups or emailed transfer logs, you can start up a _permanent_ container by adjusting the Docker parameters to run Scrappy SFTP in the background:

```
docker run -d --name sftp -p 2022:22 rhasselbaum/scrappy-sftp
```

Then you can use standard Docker commands like `docker ps` and `docker logs` to monitor the server and see real-time file transfers. But if you want to learn how to store files on the host file system, perform backups, or email transfer logs, read on.

# Preparing a container
In this section, we'll look at how to build a durable SFTP container that supports backups and emailed transfer logs.

## Data storage and backup
Scrappy SFTP enables you to store the data it maintains on the host file system, which makes it easy to backup or move the container to a different host. This data includes:

* The SFTP directory structure and files
* The user database and SSH host keys

All user directories are stored in the `/sftp-root` container volume, and the user database and SSH host keys are replicated to the `/creds` volume. To use directories on the host for both, specify their locations using the `-v` argument of `docker run`. For example:

```
docker run -d --name sftp -p 2022:22 \
 -v /host/path/to/sftp-root:/sftp-root \
 -v /host/path/to/credentials:/creds \
 rhasselbaum/scrappy-sftp
```

Scrappy SFTP will create subdirectories in the `/sftp-root` volume as users are added through the `addsftpuser` command shown earlier. It also creates socket files and hard links to log child process activity, so make sure the file system on the host allows these special file types.

The user database and SSH host keys are stored in their normal locations within the container's `/etc` directory. But as changes are made, they are periodically copied into the `/creds` volume for backup. When the container starts up, any backup files contained in the `/creds` volume will be imported back into the container's `/etc` directory, making it easy to move the container to a different host transparently.

__Note__: It may take up to 10 seconds after a change is made to the user database for the change to propagate to the `/creds` volume. _Do not shut down the container until this process completes or you may lose your change._ You can run `docker logs` to see when credentials have been exported to the volume.

You can export a fresh user database and set of SSH keys without starting the SFTP server using this command:

```
docker run --rm -v /host/path/to/credentials:/creds \
 rhasselbaum/scrappy-sftp exportcreds
```

The `/host/path/to/credentials` must be empty for this command to work correctly. Otherwise, the container will attempt to import what's there and you will end up with identical files in the export.

The ownership of the SFTP files and directories in `/sftp-root` is based on user and group IDs stored in the user database, so the contents of the `/sftp-root` and `/creds` volumes must be kept in sync. Also, since the `/creds` directory contains sensitive data such as password hashes, it is recommended that only root have access to it:

```
chmod 700 /host/path/to/credentials
```

We now have a working container that supports backups. Cool! Now let's look at transfer logs.

## Transfer logs

You can use the standard `docker logs` command to view user activity including file transfers. For example, this command will show file transfers in real-time for a container called `sftp`:

```
docker logs -f sftp
```

Scrappy SFTP can also send transfer logs via email using the built-in [SSMTP](https://packages.qa.debian.org/s/ssmtp.html) mail forwarding agent. To get this working, you need to create an `ssmtp.conf` file containing your mail server connection details and account credentials.  The [man page](http://linux.die.net/man/5/ssmtp.conf) describes the file syntax in detail and the Arch Linux wiki has an [example for Gmail](https://wiki.archlinux.org/index.php/SSMTP).

Once you've got the file created, you can enable email in the container with a couple of additional options in `docker run`.

```
docker run -d --name sftp -p 2022:22 \
 -v /host/path/to/ssmtp.conf:/etc/ssmtp/ssmtp.conf \
 -e MAIL_TO=user@example.com \
 -e 'MAIL_FROM=SFTP Daemon <sftp-daemon@example.com>' \
 rhasselbaum/scrappy-sftp
```

The `MAIL_TO` variable lets you to specify the email recipient for transfer logs, and by doing so, enables email functionality in the container. A `MAIL_FROM` value can be given to set the FROM header on generated emails. It can take a simple email address or the more expressive "Display Name &lt;user@example.com&gt;" form. Note that some mail systems (e.g. Gmail) will use the authenticated user's address regardless of what appears in the FROM header, so this variable is optional.

A transfer log is sent out after each session closes. You can use `docker logs`to troubleshoot email problems. The output of SSMTP's `sendmail` command is included there.


# Putting it all together

The following command creates a container that combines all of the features above:

```
docker run -d --name sftp -p 2022:22 \
 -v /host/path/to/sftp-root:/sftp-root \
 -v /host/path/to/credentials:/creds \
 -v /host/path/to/ssmtp.conf:/etc/ssmtp/ssmtp.conf \
 -e MAIL_TO=user@example.com \
 -e 'MAIL_FROM=SFTP Daemon <sftp-daemon@example.com>' \
 rhasselbaum/scrappy-sftp
```

These options support external storage for files and the user database, as well as emailed transfer logs.

# Limitations

Scrappy SFTP works well for a population of a few dozen to a couple hundred semi-trusted users. It partitions user data directories with chroot and sets appropriate file permissions, but it is not well-suited for completely untrusted users or high traffic volumes. Here are some limitations I'd like to address in the future:

* Per user limits on number of sessions, transfers, and storage capacity
* Retry mechanism for failed email attempts

Want to help? Pull requests welcome!