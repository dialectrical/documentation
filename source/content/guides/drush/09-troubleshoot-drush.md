---
title: Drupal Drush Command-Line Utility on Pantheon
subtitle: Troubleshoot Drush
description: Review common Drush troubleshooting scenarios.
cms: "Drupal"
categories: [get-started]
tags: [migrate, terminus, drush]
layout: guide
showtoc: true
permalink: docs/guides/drush/troubleshoot-drush
anchorid: troubleshoot-drush
---

This section provides common Drush troubleshooting scenarios.

### Reading the Pantheon Environment from Drush

Since Drush does not run using the web server, reliance on the `$_SERVER` superglobal is problematic as some of the contents of that array will be missing, `['PANTHEON_ENVIRONMENT']` in particular. Drush commands and policy files should instead reference `$_ENV` when reading Pantheon environment information. For more information, refer to our documentation on [using the `$_SERVER` superglobal in custom code](/guides/environment-configuration/read-environment-config/#using-_server).

### Terminus Drush Silent Failure

The following silent failure occurs when executing `terminus drush` commands on environments that use redirect logic without checking to see if Drupal is running via the command line:

```bash
[notice] Command: <site>.<env> -- 'drush <command>' [Exit: 1]
[error]
```

Newer versions of Drush fail with a message `[warning] Drush command terminated abnormally.`. For example: 

```bash
[warning] Drush command terminated abnormally.
[notice] Command: <site>.<env> -- 'drush <command>' [Exit: 1]
[error]
```

Redirects terminate the PHP process before the Drush execution is complete. You can resolve this error by adding `php_sapi_name() != "cli"` as a conditional statement to all redirect logic within `settings.php`:

```php:title=settings.php
// Require HTTPS, www.
if (isset($_ENV['PANTHEON_ENVIRONMENT']) &&
  ($_ENV['PANTHEON_ENVIRONMENT'] === 'live') &&
  // Check if Drupal or WordPress is running via command line
  (php_sapi_name() != "cli")) {
  if ($_SERVER['HTTP_HOST'] != 'www.yoursite.com' ||
      !isset($_SERVER['HTTP_USER_AGENT_HTTPS']) ||
      $_SERVER['HTTP_USER_AGENT_HTTPS'] != 'ON' ) {
    header('HTTP/1.0 301 Moved Permanently');
    header('Location: https://www.yoursite.com'. $_SERVER['REQUEST_URI']);
    exit();
  }
}
```

### Drush Commands on Remote Aliases Not Working from Inside Local Drupal Install

Some Drush 5 commands need to be executed from outside the context of a local Drupal installation, due to a [known issue with Drush 5](https://github.com/drush-ops/drush/issues/313). The output from a Drush 5 command run in this context looks like the following:

```bash{outputLines:2-8}
drush @pantheon.SITENAME.ENV status
 PHP configuration : /srv/bindings/754cbef0a7b54a07ab07167ef8de7377/php53.in
                        i
                        /srv/bindings/754cbef0a7b54a07ab07167ef8de7377/php53.in
                        i
 Drush version : 5.10.1
 Drush : /srv/bindings/754cbef0a7b54a07ab07167ef8de7377/drushrc.
 configuration php
```

To make your Drush 5 commands work on Pantheon aliases, change your directory to a context outside of a working local Drupal installation:

```bash{outputLines:2, 5, 6-31}
pwd
/Users/USERNAME/Sites/SITENAME
cd ..
pwd
/Users/USERNAME/Sites/
drush @pantheon.SITENAME.ENV status
 Drupal version : 7.26
 Site URI : ENV-SITENAME.pantheonsite.io
 Database driver : mysql
 Database hostname : 10.178.14.16
 Database username : pantheon
 Database name : pantheon
 Database : Connected
 Drupal bootstrap : Successful
 Drupal user : Anonymous
 Default theme : bartik
 Administration theme : seven
 PHP configuration : /srv/bindings/754cbef0a7b54a07ab07167ef8de7377/php5
                           3.ini
                           /srv/bindings/754cbef0a7b54a07ab07167ef8de7377/php5
                           3.ini
 Drush version : 5.10.1
 Drush configuration : /srv/bindings/754cbef0a7b54a07ab07167ef8de7377/drus
                           hrc.php
 Drupal root : /srv/bindings/754cbef0a7b54a07ab07167ef8de7377/code
 Site path : sites/default
 File directory path : sites/default/files
 Private file : sites/default/files/private
 directory path
 Temporary file : /srv/bindings/754cbef0a7b54a07ab07167ef8de7377/tmp
 directory path
```

### Drush Error: "Unknown option: --db-url"

```bash{outputLines:2-3}
drush @pantheon.SITENAME.ENV cc all
Unknown option: --db-url. See `drush help cache-clear` for available [error]
options. To suppress this error, add the option --strict=0.
```

To resolve this error, follow the suggestion and add the option `--strict=0`:

```bash{outputLines:2-3}
drush @pantheon.SITENAME.ENV cc all --strict=0
'all' cache was cleared in [success]
/srv/bindings/BINDINGID/code#ENV-SITENAME.pantheonsite.io
```

This only affects Drupal 7 sites running a Drush version below Drush 8

### Drush Error: "No Drupal site found", "Could not find a Drupal settings.php file", or missing system information from status

```none
Could not find a Drupal settings.php file at ./sites/default/settings.php
```

To resolve, add a default or empty `sites/default/settings.php` to your site's code.

### Unable to Connect to MySQL Server

Sometimes, you may encounter the following error when running Drush MySQL commands:

```none
ERROR 2003 (HY000): Can't connect to MySQL server on 'dbserver.dev.SITE_ID.drush.in' (61)
```

This can happen when an inactive site has spun down. To resolve this error, wake environments by loading the home page or with the following [Terminus](/terminus) command:

```bash{promptUser: user}
terminus env:wake SITENAME.ENV
```

### Unable to Connect to drush.in Hostnames (DNS)

Some ISPs have issues resolving a drush.in hostname; if you're having trouble connecting to a drush.in hostname, you can use the `dig` command to investigate further.

```bash{outputLines:2-19}
dig appserver.live.38f2bd91-0000-46cb-9278-0000000000000.drush.in
;; Truncated, retrying in TCP mode.


; <<>> DiG 9.8.1-P1 <<>> appserver.live.38f2bd91-0000-46cb-9278-0000000000000.drush.in
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: REFUSED, id: 38905
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 0


;; QUESTION SECTION:
;appserver.live.38f2bd91-0000-46cb-9278-0000000000000.drush.in. IN A


;; Query time: 11 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Thu Aug 30 12:28:25 2012
;; MSG SIZE rcvd: 78
```

As you can see in the output above, the status REFUSED suggests improper resolution. The next step is to run `dig` with a specified DNS server. We recommend using Google's DNS (8.8.8.8):

```bash{outputLines:2-26}
dig @8.8.8.8 appserver.live.38f2bd91-0000-46cb-9278-0000000000000.drush.in
;; Truncated, retrying in TCP mode.


; <<>> DiG 9.8.1-P1 <<>> @8.8.8.8 appserver.live.38f2bd91-0000-46cb-9278-0000000000000.drush.in
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 36494
;; flags: qr rd ra; QUERY: 1, ANSWER: 34, AUTHORITY: 0, ADDITIONAL: 0


;; QUESTION SECTION:
;appserver.live.38f2bd91-0000-46cb-9278-0000000000000.drush.in. IN A


;; ANSWER SECTION:
appserver.live.38f2bd91-0000-46cb-9278-0000000000000.drush.in. 599 IN A 67.207.144.213
...
appserver.live.38f2bd91-0000-46cb-9278-0000000000000.drush.in. 599 IN A 67.207.143.122


;; Query time: 52 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Thu Aug 30 13:02:00 2012
;; MSG SIZE rcvd: 622
```

In this example, Google's DNS is able to properly resolve the drush.in hostname.

You can adjust your local settings to use Google's DNS (8.8.8.8 and 8.8.4.4) instead of the default provided by your ISP to properly resolve the hostnames.

### Timeouts When Using Long-Running Migrate or Other Drush Commands

Long-running Drush commands that produce no output will cause the SSH gateway to timeout. Pantheon's timeouts for SSH based commands are outlined in our [documentation on timeouts](/timeouts). To avoid a timeout related to a lack of output, be sure your commands return output to the terminal session in under 10 minutes.

For example, using the `--feedback` flag:

```bash{promptUser: user}
drush migrate-import migration --feedback="1000 lines processed"
```

### Drush error: Failed opening required .../vendor/bin/includes/preflight.inc

```none
Fatal error: require(): Failed opening required '/srv/bindings/.../code/vendor/bin/includes/preflight.inc' (include_path='.:/usr/share/pear:/usr/share/php') in /srv/bindings/.../vendor/bin/drush.php on line 11
```

This indicates that the vendor directory contains Drush binaries that should be removed. Remove any Drush files from `vendor/bin` and `vendor/drush` using `git rm`.

## More Resources

- [Timeouts](/timeouts)
- [Domains on Pantheon Guide](/guides/domains/dns)