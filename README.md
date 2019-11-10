# Set Up SpamAssassin on Ubuntu 

### Note: This article pertains to Ubuntu 12, and it applies to versions of Ubuntu or Debian that use init rather than systemd. To install SpamAssassin on Ubuntu 15+, see the [instructions here](Ubuntu15.md).

In our previous installment, we set up a Postfix and Dovecot mail server with virtual domains and users on Ubuntu 12.04. It works really well. It works so well that everybody wants to use our new mail server to sell Viagra and cheap home loans. Enough. Let's set up the SpamAssassin anti-spam system to eliminate all the junk mail.

### Step 1: Install SpamAssassin and its client

```shell	
$ apt-get install spamassassin spamc
```

### Step 2: Add a user for the spamd daemon

```shell	
$ adduser spamd --disabled-login
```

### Step 3: Edit the configuration settings at /etc/default/spamassassin

I'll only list the values I'm changing.

```ini
# Change to one to enable the spamd daemon
ENABLED=1
 
# Set the username, home directory and log file
SPAMD_HOME="/home/spamd/"
OPTIONS="--create-prefs --max-children 5 --username spamd --helper-home-dir ${SPAMD_HOME} -s ${SPAMD_HOME}spamd.log"
 
# Set a location for the Process ID file
PIDFILE="${SPAMD_HOME}spamd.pid"
 
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

This will cause each email that scores 5.0 or greater on its spam checks to marked with ***** SPAM ***** and the score in its subject line. Most email clients already recognize the headers or text that SpamAssassin adds to incoming emails, but if you have a client that doesn't, you can set up a filtering rule to send any message with ***** SPAM ***** in its subject line directly to your junk folder.

The "bayes" rules allow the Bayesian filter to try to identify spam. And if report_safe is set to 0, incoming spam will be identified by "X-Spam-" headers added to the message while no changes are made to the message body.

### Step 5: Tell Postfix to pass incoming messages to the anti-spam system for checking

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
$ service spamassassin start
$ service postfix restart
```

Try sending your accounts on the server a couple of emails from an external account (Gmail, Outlook.<span></span>com, whatever) and check the full message headers after you've received them. You should see some entries like this if everything is working correctly:

```ini	
X-Spam-Checker-Version: SpamAssassin 3.3.2 (2011-06-06) on myserver.com
X-Spam-Level: 
X-Spam-Status: No, score=0.3 required=5.0 etc. etc.
```

I find the basic configuration above to be pretty solid, but of course you can tinker with the spam scores, filtering methods, or other settings if you like.