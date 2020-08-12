# RidffBackup-Wrapper

RidffBackup-Wrapper (RdiffBackup) is an opionated, semi thin CLI wrapper to the awesome [rdiff-backup](https://github.com/rdiff-backup/rdiff-backup) tool which provides a simplified interface. I found that using rdiff-backup native command can be pretty difficult to remember. Backups to me is something that should be easy to do, and restores even easier. On the rare occassion that I needed to recover a file I had to figure out the rdiff-backup commands to do the restore (always in a time crunch). At last count it had 74 CLI switches!

# Some benefits of using this tool
  - JSON based config and most options can be overridden via cli switches
  - Greatly simplifies the backup/restore functions
  - Human readable reports delivered via email
  - Run pre/post commands before/after any jobs start, and per backup source.
  - Supports local and remote backups
  - Set retention periods on each source
  - Verify backups
  - Battle tested for several years (~2014)

Currently the wrapper is perl based because I wrote a very long time ago. Once the rdiff-backup project re-writes the python API I might re-write this to work natively with it.

### Installation

RdiffBackup requires these dependencies to be installed (should be supported by your OS package manager)
  - [JSON](https://metacpan.org/pod/JSON)
  - [Data::Dumper](https://metacpan.org/pod/Data::Dumper)
  - [Mail::Sendmail](https://metacpan.org/pod/Mail::Sendmail)
  - [rdiff-backup](https://github.com/rdiff-backup/rdiff-backup)



On the client install all the dependencies and copy in the example config.
```sh
$ install RdiffBackup -o root -g root -m 755 /usr/local/bin/RdiffBackup
$ cp examples/RdiffBackup.conf-remote /etc
$ vi /etc/RdiffBackup.conf
$ cat /root/.ssh/id_rsa.pub
```

On the server just rdiff-backup (and sshd) also needs to be installed.
```sh
$ adduser remoteclient
$ install -d -m 700 /home/remoteclient/ssh
$ echo "ssh public key" > /home/remoteclient/.ssh/authorized_keys
$ chown -R remoteclient: /home/remoteclient/.ssh
```
You may want to move (and are encouraged to do so) these home dirs to a special mounted drive. doing do has special SELinux implications.

# Common CLI Usage
### Run a complete backup
```sh
$ RdiffBackup
```

### Run a complete backup using a different config
```sh
$ RdiffBackup --config /usr/local/etc/RdiffBackup.conf
```

### Backup before you make some changes
```sh
$ RdiffBackup --source /etc
```

### See what backups had changes
If you don't provide a source, all sources are shown
```sh
$ RdiffBackup --list /etc
--/etc
Found 584 increments:
    increments.2020-06-05T06:07:50-04:00.dir   Fri Jun  5 10:07:50 2020
    increments.2020-06-05T06:31:39-04:00.dir   Fri Jun  5 10:31:39 2020
    increments.2020-06-05T08:00:06-04:00.dir   Fri Jun  5 12:00:06 2020
    increments.2020-06-05T10:01:27-04:00.dir   Fri Jun  5 14:01:27 2020
    ...
```
### See what files changed for a backup
If you don't provide a source, then all sources are shown. --time is required
```sh
$ RdiffBackup --listchanged /etc --time 2020-06-05T06:07:50-04:00
--/etc
changed RdiffBackup.conf
new     apt/sources.list.d/buster-backports.list
changed hosts
deleted ssl/certs/116bf586.0
```
### Restore a file
Using the --dest flag to change the location or name of the file, otherwise its restored to the original location if the file does not exist. Use --force to overwrite if it exists. --time is required.
```sh
$ RdiffBackup --recover /etc/hosts --time 2020-06-05T06:07:50-04:00 --dest /etc/hosts.restored
Restoring /etc/hosts from 2020-06-05T06:07:50-04:00 to /etc/hosts.restored
Restored File /etc/hosts.restored
$ ls -la /etc/hosts.restored
-rw-r--r-- 1 root root 216 May 30 21:44 /etc/hosts.restored
```
### Clean up the backup server of backups per source
If --time is omitted the retention period used in the config file is then used.
```sh
$ RdiffBackup --runretention /etc --time 5D
Fatal Error: Found 2 relevant increments, dated:
Thu Jul 30 16:59:56 2020
Fri Jul 31 13:28:35 2020
If you want to delete multiple increments in this way, use the --force.
```

# JSON Configs
```json
{
    "hostname": "override the clients hostname",
    "bkuphost": "FQDN of the remote host",
    "emailto": "you@email.com",
    "prerun": [ "hostname", "whoami" ],
    "defaultretention": "33D",
    "lockfile": "/var/tmp/lockfile",
    "lockfileage": "86400",
    "smtpserver": "mailrelay.domain.com",
    "smtpserverport": 25,
    "sshport": 2222,
    "keyfile": "/root/.ssh/altkey",
    "sources": {
        "/etc": {
            "excludes": [ "**rpmnew", "**rpmsave", "/etc/iptables/sets" ],
            "retention": "90D",
            "verify": "1"
        },
        "/home/username": {
            "excludes": [ "**.bash_history", "**Downloads", "**tmp", "**.cache", "**.npm", "**go/pkg", "**build" ],
            "retention": "21D"
        },
        "/opt": {
            "excludes": [ "**backup" ]
        },
        "/root": {
            "excludes": [ "**.bash_history", "**.yarn", "**.cache", "**.npm", "**go/pkg" ],
        },
        "/var/www/vhosts": {
            "excludes": [ "**.log" ],
            "verify": "1"
        },
        "/usr/local/bin": {
        }
    }
}
```
## JSON keys and values explained
 - hostname
   - Used for the ssh username
   - If not specified, web-01.prd.domain.com becomes "web-01"
   - Example web-01.prd.domain.com -> web-01-prd
 - bkuphost
   - The FQDN of the **remote** backup host
   - Dont use if doing a local backup
 - back2disk
   - Sets the local path where to send backups to
   - Needs to have a file named "rdiffbackupdisk" in the root of the drive. This ensures the drive is mounted and you won't unexpectedly fill up the another drive like "/"
 - emailto
   - Sets the email "To" address
 - emailfrom
   - Sets the email "From" address
 - defaultretention
   - Sets a default retention time for all sources if not defined in each source
   - Defaults to 33 days if not set.
 - lockfile
   - File to serve as a lockfile to ensure that multiple backups don't run concurrently
   - defaults to /var/tmp/RdiffBackup.lock
 - lockfileage
   - Defines the maximum age of the lockfile. This assumes that the last RdiffBackup crashed and left a stale lockfile
 - smtpserver
   - Set the smtp relay to send the report though
   - Defaults to 127.0.0.1
 - smtpserverport
   - Sets the smtp server port to connect to
 - keyfile
   - Sets the ssh key to use
   - Default to the accounts default which the backup is running under
 - prerun
   - List of commands to run before sources are processed. If any command fails (exits non-zero), the script exits and nothing is backed up.
 - postrun
   - List of commands to run after sources are processed. No real consequence if these command fail.
 - Sources
   - Each source is an object where attributes can be set per source. The object key is the full path to the directory which will be backed up.
     - excludes
       - see the (man page)[https://rdiff-backup.net/docs/rdiff-backup.1.html] on how this works.
     - retention
       - Sets the number of "units" to keep this source of prior backups around.
       - Defaults to the global "defaultretention" if not defined
       - See the man page on (--remove-older-than)[https://rdiff-backup.net/docs/rdiff-backup.1.html] for details
     - verify
       - bool to enable or disable verification of each source.
       - defaults to 0
       - can be enabled globally if --verify is passed in on the CLI
     - rdiffopts
       - Allows you to set custom rdiff-backup options.
       - Default is "--exclude-sockets --exclude-fifos --exclude-device-files --create-full-path"
     - prerun
       - Run this command before attempting a backup
       - If the command failed (non-zero exit code) then the source is skipped unless overridden
     - ignoreprefail
       - bool to force a backup to run if the "prerun" command fails
       - defaults to 0
     - postrun
       - Run this command after the source has been backed up.
     - ignorepostfail
       - Ignores postrun commands that fails
