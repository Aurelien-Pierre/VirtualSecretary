# Virtual Secretary

## What is the Virtual Secretary ?

Imagine a world where :

* emails get tagged "urgent" if your agenda records an appointment in less than 48h with the email sender,
* terribly long emails get tagged "call me" if you have the phone number of their senders in your contact book,
* emails with `.ics` calendar events invites get tagged "RSVP"
* emails get sorted in folders by the CardDAV category in which their sender is, so your college alumni don't get mixed up with your family,
* emails sent by people into your customer SQL database automatically get a "client" label,
* emails sent by bulk-emailing systems get moved out of the way to their own folder, for you to read when you have time,
* etc.

The Virtual Secretary connects to :

* your IMAP email server (incoming emails),
* your SMTP email server (outgoing emails),
* your CardDAV contacts server,
* your CalDAV agendas server,
* your MySQL/MariaDB databases server.

It then provides you with an high-level Python interface to write mail filters that can cross-check data from emails, contacts, agenda and databases and define actions like (un)tagging/moving/deleting emails, updating contacts or appointments data, etc. It can also define auto-responders, email digests, etc.

Aside of the simple filter use cases, it lets you use the full extend of Python scripting to use machine-learning modules, regular expressions pattern matching, or even write your own data connectors.

It comes with a large range of example filters that can be used as-is or as templates, including a ready-to-use anti-spam system using IP blacklist/whitelist and neural network AI that can be trained against your own spam box.

The functionnal logic is very similar to the one of [IFTTT](https://ifttt.com/explore/new_to_ifttt), plus the whole Python ecosystem to extend actions, minus the toy GUI, without the anxiety of SaaS (shit as a software) owned by some company that may go out of business or extinct the product (wink wink [Yahoo! Pipes](https://en.wikipedia.org/wiki/Yahoo!_Pipes)).


## How it works

The `main.py` scripts needs to be called with the path to a configuration directory, like `python main.py ~/secretary/config process`. Say you have 2 email addresses, `me@domain.com` and `pro@domain.com`, you need to create a folder for each email plus one `common` folder in your `config` directory. Then populate them with your filters, named like `00-protocol-filter.py` and your credentials into a `settings.ini` configuration file. This gives you :

* `/config`:
  * `/common`:
    * `00-imap-spam.py`
    * `01-imap-urgent.py`
  * `/me`:
    * `settings.ini`
    * `03-imap-family.py`
    * `04-imap-banks.px`
  * `/pro`:
    * `settings.ini`
    * `03-imap-colleagues.py`
    * `04-imap-clients.py`

The filters put in `/common` folder will be used for all email addresses. The filters put in the email folder will be private to this email account. The `/common` folder needs to have this exact name, but the email folders can be named whatever you like and will be run in alphabetical order.

Filters will be processed in the order defined by their 2-digits priority prefix, from `00` to `99`. A `LEARN` prefix can be used instead of the 2 digits to define special filters that collect data in a read-only fashion and may use heavy processing. If the same priority level is found in both the global `common` folder and a local email folder, the global filter is overriden by the local one with a warning issued in terminal. Priorities are shared between protocols.

The script will process all the filters it finds in all the folders it finds, so you just need to create folders and filters.

The protocol that triggers the filter needs to be written in the filter name just after the priority. The available choices are `imap`, `carddav`, `caldav`, `mysql`. Even though filters can connect different protocols, one of them needs to be the input signal that will trigger the whole filtering, so for example, the `imap` trigger will launch the filters for each email in some mailbox and then dispatch events to other accounts.

The `settings.ini` need to contain the login credentials for at least the trigger protocol of each filter, like :

```ini
[imap]
user = me@server.com
password = XXXXXXXX
server = mail.server.com
entries = 20

[smtp]
user = me@server.com
password = XXXXXXXX
server = smtp.server.com
```

Filters are defined like such: in a `01-imap-invoice.py` file, write :

```python
#!/usr/bin/env python3

protocols = globals()
secretary = locals()
imap = protocols["imap"]
filtername = secretary["filtername"]

def filter(email) -> bool:
  return "invoice" in [email.body["text/plain"].lower(), email.header["Subject"].lower()]

def action(email):
  email.move("INBOX.Invoices")

imap.get_mailbox_emails("INBOX", n_messages=20)
imap.run_filters(filter, action, filternamer, runs=1)
```

This very basic filter will fetch the 20 most recent emails (`n_messages=20`) from the mailbox `INBOX` (the base IMAP folder), look for the case-insensitive keyword "Invoice" in the email body and subject, and if found, will move the email to the "Invoices" IMAP folder. This folder will be automatically created if it does not exist already. The filter will be run at most one time (`runs=1`) for each email, which means that if you manually move the emails back to `INBOX` after the filter is applied, the next run will not move them again to "Invoices". The history of runs is stored in an hidden log file named `.01-imap-invoice.py.log` where emails are identified globally for all the folders in the email account, and the history is kept even if you change the content of the filter in the future.

To run all the typical filters (prefixed with 2 digits and processing data read-write) in sequence, for all subfolders in the config directory, call :
```bash
python src/main.py /path/to/config process
```

To run all the learning filters (prefixed with `LEARN` and processing data read-only) in sequence, for all subfolders in the config direcotry, call :
```bash
python src/main.py /path/to/config learn
```

You may find several example filters in the `/examples` folder, along with the source code.

When you run the `main.py` script, it will create a `sync.log` file into each email folder logging all operations applied on emails, connection statuses and available IMAP folders. This will help writing and debugging your filters. Errors and exceptions of the program are logged in the standard output of the terminal you used to launch the main script.

## Installing

1. Install Git and Python 3:
   1. On Debian, Ubuntu and their derivatives (Mint, etc.): `$ sudo apt install git python3`
   2. On Redhat and all its derivatives (Fedora, CentOS, etc.): `$ sudo dnf install git python3`
2. Clone this repository: `$ git clone https://github.com/Aurelien-Pierre/VirtualSecretary.git`
3. Get into the directory: `$ cd VirtualSecretary`
4. Copy the `examples` to your own `config` folder : `cp examples config`. This is important because updates may erase your changes within the `examples` folder.
5. Edit the content of the `settings.ini` within `config/me@server.com` with your own credentials
6. Run the learning stage: `$ python src/main.py config/ learn`
7. Run the filtering stage: `$ python src/main.py config/ process`

To run the filters in background, for example on a server, you may use a cron job :

1. Start the cron editor: `$ crontab -e`
2. To execute the Virtual Secretary processing stage every 10 minutes (starting on flat hours), write the following rule:
`$ */10 * * * * python /home/your_user/path/VirtualSecretary/src/main.py /home/your_user/path/VirtualSecretary/config process &>> /home/your_user/path/VirtualSecretary/config/secretary.log`
3. To execute the Virtual Secretary learning stage every day at 0h05 and 12h05, write the following rule:
`$ 5 0,12 * * * python /home/your_user/path/VirtualSecretary/src/main.py /home/your_user/path/VirtualSecretary/config learn &>> /home/your_user/path/VirtualSecretary/config/secretary.log`
4. Save.

The `&>> secretary.log` part will save the runtimes, errors and warnings of the program to a `secretary.log` file inside your config directory. This may contain useful information to troubleshoot your installation. The program will also create a `sync.log` file in each subfolder, containing the report of all operations performed on the triggers (like email moved where, tagged how, etc.).

Note that a locking mechanism prevents 2 instances of the Virtual Secretary to process the same subfolder at the same time. If the previous run did not end before the next one starts, the next one will be aborted. When you run `$ python src/main.py` (with the proper arguments, see above), it will give you the global execution time at the end. Your delay between 2 cron jobs needs to be at least this time. This is also why we start the learning stage 5 minutes off the flat hour in the example above.


## Updating

1. Get into the directory: `$ cd /path/to/VirtualSecretary`
2. Run Git pull: `$ git pull`
