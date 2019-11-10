# Set Up SpamAssassin On Ubuntu 15+

Note: This article describes setting up SpamAssassin 3 to work with a Postfix/Dovecot email server on Ubuntu 16, and it applies to other Debian/Ubuntu variants that use systemd. There are some significant differences in the SpamAssassin config between distros that use systemd and those that use upstart. For the Ubuntu 12/14 version of this article, [go here](README.md).

### Step 1: Install SpamAssassin and its client
	
```shell
$ apt-get install spamassassin spamc
```

### Step 2: Add a user for the spamd daemon
	
```shell
$ adduser spamd --disabled-login
```

### Step 3: Edit the configuration settings at /etc/default/spamassassin

My comments here are marked with the ## symbol. systemd does its own shell-style parsing, so we need to change some of our directives from before.

```ini
## With the arrival of systemd, the ENABLED parameter is now ignored. You'll enable SpamAssassin in step 6, below.
# ENABLED=0
 
# Set the username, home directory and log file
## A change from before here as well. Previously we defined a SPAMD_HOME variable and inserted it 
## via brackets{} to target the /home/spamd/ directory. However, systemd no longer expands these 
## variables, causing ${SPAMD_HOME} to be printed verbatim. We're using the actual directories here instead.
OPTIONS="--create-prefs --max-children 5 --username spamd --helper-home-dir /home/spamd/ -s /var/log/spamassassin/spamd.log"
 
## I'm also setting the logging to /var/log to make it subject to normal log rotation. 
## Additional issue here: spamd needs to restart after log rotation (see the manpage entry for -s at 
## https://spamassassin.apache.org/full/3.4.x/doc/spamd.html). 
## Hack: just add `systemctl restart spamassassin` to the end of /etc/cron.daily/logrotate .
 
# Set a location for the Process ID file
## We used SPAMD_HOME here before as well.
## Default is now /var/run/spamassassin.pid set in the /lib/systemd/system/spamassassin.service file. 
## There's no particular reason to change it, but if you do, remember to do `systemctl daemon-reload` before starting spamd.
# PIDFILE="/var/run/spamd.pid"
 
# Set the sa-update process to update the anti-spam rules automatically on a nightly basis
CRON=1
```

### Step 4. Edit /etc/spamassassin/local.cf to set up some anti-spam rules

I use the following settings:

```ini	
rewrite_header Subject ***** SPAM _SCORE_ *****
report_safe             0
required_score          5.0
use_bayes               1
use_bayes_rules         1
bayes_auto_learn        1
skip_rbl_checks         0
use_razor2              0
use_dcc                 0
use_pyzor               0
```

This will cause each email that scores 5.0 or greater to be marked with ***** SPAM ***** and the score in its subject line. Most email clients already recognize the headers or text that SpamAssassin adds to incoming emails, but if you have a client that doesn't, you can set up a filtering rule to send any message with ***** SPAM ***** in its subject line directly to your junk folder.

The "bayes" rules allow the Bayesian filter to try to identify spam. And if report_safe is set to 0, incoming spam will be identified by "X-Spam-" headers added to the message while no changes are made to the message body.
Step 5: Tell Postfix to pass incoming messages to the anti-spam system for checking

Edit /etc/postfix/master.cf and add a content filter to your SMTP server:

```ini
smtp      inet  n       -       -       -       -       smtpd
        -o content_filter=spamassassin
```

Still in /etc/postfix/master.cf, add this to the end of the file:

```ini	
spamassassin unix -     n       n       -       -       pipe
        user=spamd argv=/usr/bin/spamc -f -e  
        /usr/sbin/sendmail -oi -f ${sender} ${recipient}
```

This tells the client to pass its completed checks to the default MTA (mail transfer agent, /usr/sbin/sendmail, i.e., the Postfix-Sendmail compatibility interface) for handoff to the MDA (mail delivery agent, Dovecot) for delivery.

### Step 6: Start/restart everything and test

```shell	
$ systemctl restart postfix.service
$ systemctl enable spamassassin.service
$ systemctl start spamassassin.service
```

(To check SpamAssassin's startup status, do systemctl list-units --type=service --all. spamassassin.service will be listed as active/loaded if it's running properly.

```ini	
spamassassin.service                       loaded    active     Perl-based spam filter using text analysis
```

Now try sending your accounts on the server a couple of emails from an external account (Gmail, Outlook.<span></span>com, whatever) and check the full message headers after you've received them. You should see some entries like this if everything is working correctly:

```plaintext	
X-Spam-Checker-Version: SpamAssassin 3.4.1 (2015-04-28) on myserver.com
X-Spam-Status: No, score=-1.9 required=5.0 tests=BAYES_00, etc. etc.
```

I find the basic configuration above to be pretty solid, but of course you can tinker with the spam scores, filtering methods, or other settings if you like.