# Three — Hack The Box

> Starting Point, Tier 1. Linux.

## Summary

A static site that gives up nothing on the main host, because the application is
not there. The real target is a virtual host whose name is short enough that
most wordlists miss it, and that host turns out to be an S3 bucket with
anonymous write access sitting in front of the web root. Uploading a PHP shell
into the bucket is the whole foothold. The box is a good lesson in two things
that look unrelated until they are not: DNS enumeration, and cloud storage
permissions treated as a web server.

## Enumeration

Port scanning on this box told me less than DNS did. The useful work was
virtual-host discovery against `thetoppers.htb`:

```
gobuster vhost -u http://thetoppers.htb \
  -w <subdomain-wordlist> --append-domain
```

The result was `s3.thetoppers.htb`, which is a hint in itself: the hostname is
the service, so I stopped treating it as a website and treated it as storage.

I did not record which wordlist I used, which is the one thing about this box I
would fix in my notes. `s3` is a three-character name and standard subdomain
lists are built from names people register, not the names people configure, so
the hit was partly luck. A short manual list of infrastructure names — `s3`,
`cdn`, `dev`, `staging`, `internal`, `assets` — is a faster second pass than
doubling the wordlist size.

## Dead ends

The main host was a static page with no links worth following, and I spent time
on it before accepting that enumeration had to move to the hostname layer.
Directory brute forcing a site that has no directories is a way to burn twenty
minutes.

The other dead end was wordlist selection. A small list returns nothing here
because the name is too short to appear, so the answer looked like "no vhosts"
when the truth was "wrong list".

## Foothold

`s3.thetoppers.htb` answered as an S3 endpoint, so I pointed `awscli` at it
directly rather than trying to browse it:

```
aws --endpoint-url http://s3.thetoppers.htb s3 ls
```

The bucket was named `thetoppers.htb` and it accepted an unauthenticated PUT, so
I uploaded a one-line shell to its root:

```
aws --endpoint-url http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb/shell.php
```

The bucket root is the document root, so Apache served it straight back:

```
curl "http://thetoppers.htb/shell.php?cmd=id"
```

That returned `www-data`, which is the whole foothold. No exploit, no CVE. A
storage bucket with the wrong policy was the vulnerability.

## Privilege escalation

None needed. The flag was readable as `www-data` at `/var/www/flag.txt`, and
the shell could read it directly.

That is worth saying plainly rather than dressing up: this box is not about
escalation. The chain ends at the misconfiguration, and the interesting question
is why the bucket allowed the write in the first place.

## What I would do differently

- Record the exact wordlist and command for every box. "Some subdomain list"
  makes a writeup unreproducible, including for me.
- Test bucket write permissions on purpose as soon as an endpoint answers, with
  a harmless `aws s3 cp /dev/null s3://bucket/test` rather than a shell.
- Stop earlier on static hosts. If the page has no forms, no parameters and no
  directories, the application is somewhere else.

## Detections

- An S3 bucket policy allowing unauthenticated `PutObject` is a reviewable
  configuration error, and the kind of thing cloud security posture tools flag
  as critical by default. It should never reach production.
- `PutObject` from an address the application never uses is a clean signal.
  CloudTrail data events are off by default, which is why this stays quiet.
- A new `.php` file in a directory that should only hold static assets should
  trip a file-integrity monitor. The webshell request itself
  (`shell.php?cmd=`) is a trivial signature once logging exists.

---

### Publishing checklist

- [x] Machine is retired or Starting Point
- [x] No flags, hashes, or cracked credentials
- [x] No target IPs included
- [x] Written in my own words
