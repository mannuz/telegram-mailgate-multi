# telegram-mailgate

telegram-mailgate is a minimalistic content filter for Postfix that sends e-mails via Telegram.
It is inspired by [gpg-mailgate](https://github.com/uakfdotb/gpg-mailgate) and is useful, for example, to receive CRON outputs directly to your mobile.

## Features

- Forwards e-mails to Telegram users, groups, or channels via a bot.
- **Supports multiple Telegram recipients per e-mail address (alias).**
- **Sends everything in a single Telegram message when using `--simple-header`.**
- Easy integration with Postfix as a content filter.


## Usage

```sh
telegram-mailgate.py [-h] [--config CONFIG] [--from FROM]
                     [--queue-id QUEUE_ID] [--raw | --simple-header]
                     to [to ...]
```

**Positional arguments:**

- `to` — the recipient(s) as defined in your alias file.
You can specify one or more.

**Optional arguments:**

- `-h, --help` — show help message and exit
- `--config CONFIG` — use a custom configuration file
- `--from FROM` — overwrite "From" header (ignored if `--raw` is specified)
- `--queue-id QUEUE_ID` — specify the queue ID of the message for logging
- `--raw` — include all sections, header, and attachments
- `--simple-header` — include a header (sender and server) in the same message as the mail content


## Alias File

The alias file (configured in `main.cf` as `core.aliases`) maps e-mail addresses to one or more Telegram chat IDs.
**Multiple chat IDs can be specified per line, separated by spaces or commas.**

**Example:**

```
[email protected] 123456789,987654321
[email protected] 555555555
```

Spaces or commas are accepted as separators.
**No comments or empty lines are allowed.**

## Installation

```sh
sudo apt update && sudo apt install python3 python3-pip
sudo adduser --system telegram-mailer  # Create a specific user to run telegram-mailgate
sudo -u telegram-mailer pip3 install --user python-telegram-bot
sudo cp telegram-mailgate/telegram-mailgate.py /usr/local/bin/
sudo chown telegram-mailer /usr/local/bin/telegram-mailgate.py
sudo chmod 700 /usr/local/bin/telegram-mailgate.py
sudo mkdir /etc/telegram-mailgate
sudo cp telegram-mailgate/{main.cf,logging.cf,aliases} /etc/telegram-mailgate/
```

Edit `/etc/telegram-mailgate/main.cf` to configure your Telegram API key.
Edit `/etc/telegram-mailgate/aliases` to configure the mapping between recipient addresses and Telegram chat IDs.

## Postfix Configuration

In `/etc/postfix/master.cf`, add the following process:

```conf
# =======================================================================
# telegram-mailgate
telegram-mailgate unix -     n       n       -        -      pipe
  flags= user=telegram-mailer argv=/usr/local/bin/telegram-mailgate.py --simple-header --queue-id $queue_id $recipient
```

Make sure the specified user has permissions to execute telegram-mailgate and that all Python dependencies are installed.

In `/etc/postfix/main.cf` add telegram-mailgate as a content filter:

```conf
content_filter = telegram-mailgate
```

Restart postfix:

```sh
sudo postfix reload
```


## Testing

```sh
echo "This is the body of the email" | mail -s "This is the subject line" <my_recipient>
```

Replace `<my_recipient>` with an address listed in your alias file.
If using a local user, you may need to add `@<hostname>` at the end.
Logs can be viewed with:

```sh
sudo tail -f /var/log/mail.log
```


## Notes

- If a single e-mail address in your alias file maps to multiple chat IDs, **the same message will be sent to all listed Telegram recipients**.
- When using `--simple-header`, **the sender and server name are included in the same Telegram message as the e-mail content**.

**Example output in Telegram:**

```
New mail on <server_name> from <sender_email>

<mail content>
```
