---
layout: post
title: Sup?
---

In this post I'm going to show you how I use [sup][] to manage multiple
emails, along side other devices.

*Note*: This is my very first long-ish technical article. Apologies if
it gets hard to follow.

[sup]: http://supmua.org/

How I Use Email
---------------

- I use both Gmail and my school email separately.
- My school email is a Google Apps account, but handles logins itself
  through OAuth2.
- My main machine that I email from is a Macbook.
- I often use email on my phone, which means changes need to appear on
  all clients.
- I prefer emails to be threaded into a single view conversation, if possible.
- I sometimes triage using labels.
- I don't organize mail into folders.
- Once done with an email thread, I archive it.
- I do not delete emails, so everything is archived and searchable.
- Single inbox for all accounts.
- I keep local backups of my email.
- Plain-text emails or die.

If this sounds vaguely like how you operate, then read on. Otherwise,
I'd suggest looking elsewhere, as this will be wasted reading for you.

*But why not just use Thunderbird/Mail.app/etc?* Because they suck.
Thunderbird once deleted all the emails out of my archive on Gmail.
Mail.app doesn't remember mail behavior settings (e.g, don't move that
to the Trash on the server on delete).

*But why not just use Mutt?* Because I can. I also like the flow of sup
a lot more than Mutt, even though they are similar.

Overview
--------

There are a few things we will need before we can get this monstrosity
working. It might even make your head hurt a bit. Because I primarily
use email on my phone and OS X, this guide will be targeted towards OS
X use (although my Arch Linux set up is nearly identical, only differing
by package managers for installing tools).

OAuth2
------

For my setup, I cannot use the ordinary username/password setup for
emails that most guides you will find. My school gives us an email that
is a Google Apps mail, but handles logins themselves. So, I am required
to use OAuth2 to login. If you don't need this, just skip this section
if you want.

### Registering your "app"

To get OAuth2 working, first we need to register an app with Google to
use their API. Don't worry, it's free, just don't give your tokens out
to other people (just in case). Head over to Google's [Developers Console][]
and create a new project. Select that project, and on the left go to
Settings > Rename project. Just give it some name you fancy.

Afterwards, go to APIs & auth > Credentials. Make note of the *Client ID*
and *Client Secret*. You'll need those for your configurations. Now we need
to authorize this app to manage your email. Easiest way is to follow
[this guide][]. Here's the summary:

1. Download [oauth2.py][]
2. Then, execute this and follow the directions:

        $ python oauth2.py --generate_oauth2_token \
        --client_id=<your_client_id> \
        --client_secret=<your_client_secret>

Make note of the *Refresh Token*, we will need that for configuration
also. Do this for each account you need, making note of which account is
related to which refresh token.

With those three things, you are armed and ready to rock with OAuth2.

[Developers Console]: https://cloud.google.com/console/project
[this guide]: https://code.google.com/p/google-mail-oauth2-tools/wiki/OAuth2DotPyRunThrough
[oauth2.py]: http://google-mail-oauth2-tools.googlecode.com/svn/trunk/python/oauth2.py


Getting email
-------------

For email, we are going to use OfflineIMAP to fetch things for us and
keep our email in sync.

### Installing

Since I require the use of OAuth2, I have a branch of [OfflineIMAP][]
going you can use until it is merged into master. If you don't need
OAuth2, just install it through your package manager.

    git clone https://github.com/cscorley/offlineimap.git
    cd offlineimap
    git checkout gmail-oauth
    python setup.py install

[OfflineIMAP]: https://github.com/cscorley/offlineimap/tree/gmail-oauth

### Configuration

Create a file `~/.offlineimaprc`. The following configuration sets up
OAuth2, so if you want to put in plain-text username and password, find
another guide.

    [general]
    ui = ttyui
    accounts = gmail,crimson
    fsync = False

    [Account gmail]
    localrepository = gmail-local
    remoterepository = gmail-remote
    status_backend = sqlite

    [Account crimson]
    localrepository = crimson-local
    remoterepository = crimson-remote
    status_backend = sqlite

    [Repository gmail-local]
    type = Maildir
    localfolders = ~/.mail/cscorley-gmail.com
    nametrans = lambda folder: {
                                'archive': '[Gmail]/All Mail'
                                }.get(folder, folder)

    [Repository gmail-remote]
    maxconnections = 1
    type = Gmail
    remoteuser = cscorley@gmail.com
    oauth2_client_id =
    oauth2_client_secret =
    oauth2_refresh_token =
    sslcacertfile = /usr/local/opt/curl-ca-bundle/share/ca-bundle.crt
    realdelete = no
    nametrans = lambda folder: {
                                '[Gmail]/All Mail':  'archive'
                                }.get(folder, folder)

    folderfilter = lambda folder: folder not in ['[Gmail]/Trash',
                                                '[Gmail]/Important',
                                                '[Gmail]/Spam',
                                                '[Gmail]/Drafts',
                                                '[Gmail]/Sent Mail',
                                                '[Gmail]/Starred',
                                                ]

    [Repository crimson-local]
    type = Maildir
    localfolders = ~/.mail/cscorley-crimson.ua.edu
    nametrans = lambda folder: {
                                'archive': '[Gmail]/All Mail'
                                }.get(folder, folder)

    [Repository crimson-remote]
    maxconnections = 1
    type = Gmail
    remoteuser = cscorley@crimson.ua.edu
    oauth2_client_id =
    oauth2_client_secret =
    oauth2_refresh_token =
    sslcacertfile = /usr/local/opt/curl-ca-bundle/share/ca-bundle.crt
    realdelete = no
    nametrans = lambda folder: {
                                '[Gmail]/All Mail':  'archive'
                                }.get(folder, folder)


    folderfilter = lambda folder: folder not in ['[Gmail]/Trash',
                                                '[Gmail]/Important',
                                                '[Gmail]/Spam',
                                                '[Gmail]/Drafts',
                                                '[Gmail]/Sent Mail',
                                                '[Gmail]/Starred',
                                                ]

Note that we are only interested in syncing two folders: `[Gmail]/All
Mail` and `INBOX`. We can just filter out the rest, they aren't
important to us. When we send a mail, Gmail will save it to Sent Mail
*and* All Mail for you. Sup has it's own internal sent mail folder.

Another point of interest is the `sslcacertfile=` field on both remote
Repositories. OS X doesn't come with one handy, so install the
curl-ca-bundle package from homebrew: `brew install curl-ca-bundle`.
On Arch Linux, it should be in the `ca-certificates` package (I think).

Finally, OAuth2! Under each remote, put your client id, client secret,
and refresh token. Boom, done.

### Running

Go ahead and run offlineimap so we can get the emails downloaded. It will
probably take awhile if you have a lot of mails:

    offlineimap

Later, we'll configure sup to run OfflineIMAP for us when we check for
new mail.

Sending email
-------------

Ahhhhh, sending email. What good is email if we can't annoy people with it?
I haven't found a client that seemed easy enough for me to extend OAuth2
into, and really I didn't have to. Python has `smtplib` and it handles
just about everything for me anyway.

### Installing

Download a copy of my [send.py][] if you're using OAuth2 and stick it
somewhere in your `$PATH` (I recommend `~/bin`). Password users can
install something like [msmtp][].

    mkdir -p ~/bin
    wget https://raw.github.com/cscorley/send.py/master/send.py -O ~/bin/send.py
    chmod +x ~/bin/send.py

Make sure that `~/bin` is in your `$PATH` environment variable.
Although, this won't matter much to us, as we will tell sup directly
where the `send.py` file is.

[send.py]: https://github.com/cscorley/send.py
[msmtp]: https://wiki.archlinux.org/index.php/Msmtp


### Configuration

Configuring this will hopefully be more straightforward. Just plop
in your id, secret, and token and bust an air guitar solo.

    [oauth2]
    request_url = https://accounts.google.com/o/oauth2/token
    client_id =
    client_secret =

    [account gmail]
    username = cscorley@gmail.com
    refresh_token =
    address = smtp.gmail.com
    port = 465
    use_ssl = True
    use_tls = False

    [account crimson]
    username = cscorley@crimson.ua.edu
    refresh_token =
    address = smtp.gmail.com
    port = 465
    use_ssl = True
    use_tls = False

*weedlywooooooowwooaaahhh*

### Running

Test your email by doing this (change the email addresses first, silly):

    echo "From: Test <test@example.com>
    To: Taco Lover <taco@gmail.com>
    Subject: Test message

    Body would go here
    " | send.py --readfrommsg


Hopefully it'll go through without a hitch.

Sup
---

Finally, we are ready to actually look at some email! Maybe OfflineIMAP
has finished syncing by now. :)

### Installing

This is going to be a bit complicated. OS X ships with Ruby 2.0, and sup
runs on Ruby 1.9.3 right now. You can install this however you like,
just as long as you're running 1.9.3.

#### Getting 1.9.3

I recommend using the [ruby-install][] tool to install different rubies and
the [chruby][] tool to switch out between them. Install them both from
your package manager:

    brew install ruby-install
    brew install chruby

After installing those, install Ruby 1.9.3 and switch to it:

    ruby-install ruby 1.9.3
    chruby 1.9.3
    ruby --version

This should print out that you are on 1.9.3. Next, we actually install
sup.

[ruby-install]: https://github.com/postmodern/ruby-install
[chruby]: https://github.com/postmodern/chruby

#### Install sup

Install the `sup` gem:

    chruby 1.9.3
    gem install sup

Done.

### Configuration

Typically at this point you would run `sup-config`. Go ahead and do
that, and when it asks you about adding sources, don't. We will
configure that ourselves.

#### Basic configuration

There should be a new directory made called `~/.sup`, and go ahead and
`cd` into it and edit the `config.yaml`. Here's mine if you skipped running
`sup-config`:

    ---
    :accounts:
    :default:
        :name: Christopher Corley
        :email: cscorley@gmail.com
        :alternates:
          - cscorley@crimson.ua.edu
        :hidden_alternates: []
        :sendmail: /Users/cscorley/bin/send.py --readfrommsg
        :signature: /Users/cscorley/.signature
        :gpgkey: ''
    :editor: /usr/local/bin/vim
    :thread_by_subject: false
    :edit_signature: false
    :ask_for_from: false
    :ask_for_to: true
    :ask_for_cc: true
    :ask_for_bcc: false
    :ask_for_subject: true
    :account_selector: true
    :confirm_no_attachments: true
    :confirm_top_posting: true
    :jump_to_open_message: true
    :discard_snippets_from_encrypted_messages: false
    :load_more_threads_when_scrolling: true
    :default_attachment_save_dir: ''
    :sent_source: sup://sent
    :archive_sent: true
    :poll_interval: 300
    :wrap_width: 0
    :slip_rows: 0
    :col_jump: 2
    :stem_language: english
    :sync_back_to_maildir: true

Most of this is preference, but there are two lines that are important:
sending and syncing mail.

First, make sure under your default account you have `:sendmail:` set
you use your preferred SMTP client. Here mine uses the `send.py` from
earlier.

Second, enable `:sync_back_to_maildir:`. This will allow sup to change
our `~/.mail`, and in turn, allowing OfflineIMAP to sync our changes back
to the server.

Now, if you try to run `sup` again, it should complain. Just do as it says
and run `sup-sync-back-maildir`.

#### Adding sources

Now time to add some sources. Start a new file at `~/.sup/sources.yaml`
and put in the following:

    ---
    - !supmua.org,2006-10-01/Redwood/Maildir
      uri: maildir:/Users/cscorley/.mail/cscorley-gmail.com/INBOX
      usual: true
      archived: false
      sync_back: true
      id: 1
      labels:
        - gmail
    - !supmua.org,2006-10-01/Redwood/Maildir
      uri: maildir:/Users/cscorley/.mail/cscorley-crimson.ua.edu/INBOX
      usual: true
      archived: false
      sync_back: true
      id: 3
      labels:
        - ua
    - !supmua.org,2006-10-01/Redwood/Maildir
      uri: maildir:/Users/cscorley/.mail/cscorley-gmail.com/archive
      usual: true
      archived: true
      sync_back: true
      id: 2
      labels:
        - gmail
    - !supmua.org,2006-10-01/Redwood/Maildir
      uri: maildir:/Users/cscorley/.mail/cscorley-crimson.ua.edu/archive
      usual: true
      archived: true
      sync_back: true
      id: 4
      labels:
        - ua
    - !supmua.org,2006-10-01/Redwood/SentLoader {}

Each account I have has two sources: The first is the `INBOX`, which is
the Maildir folder you will be working out of primarily. The second is
the `archive`, which sup can use for searching through your old emails
and display threads correctly.

#### Polling sources

Create a file `~/.sup/hooks/before-poll.rb`. This file will be ran each
time before sup checks for new mail:

    if (@last_fetch || Time.at(0)) < Time.now - 120
      say "Running offlineimap..."
      # only check non-auto-archived sources on the first run
      log system "offlineimap", "-o", "-u", "quiet"
      say "Finished offlineimap run."
      @last_fetch = Time.now
    end

This way, if offlineimap hasn't been ran in approximately two minutes,
it will sync our Maildir for us. Nice.

### Running

Run `sup` by:

    chruby 1.9.3
    sup

You should be greeted with your inbox. I recommend just putting the following
alias in your `.bashrc` or `.zshrc`: `alias sup='chruby ruby-1.9.3 && sup'`

Using Sup
---------

For the most part, sup usage in this setup is pretty close to what's
described in the [New User Guide]. There is one variation, however.
Archiving.

In sup, we won't archive anything. Gmail and OfflineIMAP handles that
for us. All we have to do is remove emails from the `INBOX` so our
changes show up across devices. To do this, we will mark mails in the
inbox for deletion when we are done. Don't worry, they are still in
Gmail's All Mail, and in turn, our local `archive` as well.

There is one way archiving can be used: it can be used to clean up your
sup inbox during triage. Label an email `todo` and archive it. Don't
worry, it's still in the inbox on Gmail, but now hidden under `todo` in
sup. The single drawback to using sup right now is labels don't sync
back to Gmail. Once you've finished with the mail, mark it for deletion.

Sorry, I don't have a way to label things both in the archive and inbox
(yet). Fortunately for me, once I am done with an email, I don't care
what it's labels were any longer (yes, I realize I'm throwing out one of
the best parts about sup).

Need to move something from the archive back into sup's inbox? Just
press `a` to unarchive it again. Boom, done. Yep, the inbox is just
a label, and archiving things just removes that label. These changes
won't, however, show up across devices.

[New User Guide]: https://github.com/sup-heliotrope/sup/wiki/New-User-Guide


Contacts
--------

todo.

Conclusion
----------

Ugh, email. Hopefully one day sup will be at the point that it
interfaces with Maildirs better, and the `INBOX`/`archive` situation won't
be so convoluted. Until then, this is the best I've come up with. So,
that's what's up.
