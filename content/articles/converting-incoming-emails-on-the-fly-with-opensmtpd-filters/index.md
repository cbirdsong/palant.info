---
categories:
- opensmtpd
- email
date: 2023-03-08T11:22:25+0100
description: OpenSMTPD filters can be used e.g. to insert a more convenient representation
  of the attachment into incoming emails. This article documents the approach.
lastmod: '2023-03-19 21:33:39'
title: Converting incoming emails on the fly with OpenSMTPD filters
---

This little adventure began with me being annoyed at DMARC aggregate reports. My domain doesn’t have enough email traffic to justify routing DMARC emails to some third-party analytics service, yet I want to take a brief glance at them. And the format of these emails makes that maximally inconvenient: download the attachment, unpack it, look through some (always messy but occasionally not even human-readable) XML code. There had to be a better way.

This could have been a Thunderbird extension, processing the email attachment in order to produce some nicer output. Unfortunately, Thunderbird extensions no longer have this kind of power. So I went for another option: having the email server (OpenSMTPD) convert the email as it comes in.

Since I already had [the implementation details of OpenSMTPD filters](/2020/11/09/adding-dkim-support-to-opensmtpd-with-custom-filters/) figured out, this wasn’t as complicated as it sounds. The resulting code is [on GitHub](https://github.com/palant/dmarc2html/) but I still want to document the process for future me and anyone else who might have a similar issue.

{{< toc >}}

## OpenSMTPD filter basics

OpenSMTPD filters are persistent processes, communicating with the email server via a [proprietary text-based protocol](https://man7.org/linux/man-pages/man7/smtpd-filters.7.html). The filter can register for any number of events. In case of `report` events, the filter will simply receive a notification. For `filter` events, the filter can respond with an action, e.g. telling the email server to reject an email as spam. One particularly powerful event is `data-line`, allowing the filter to handle and potentially modify the lines of incoming emails.

Previously, I’ve already written Python-based filters to [add DKIM support to OpenSMTPD](/2020/11/09/adding-dkim-support-to-opensmtpd-with-custom-filters/). The `opensmtpd.py` library I wrote there handles all the details of the communication protocol. So a filter looks like this:

```python
from opensmtpd import FilterServer

def start():
    server = FilterServer()
    server.register_message_filter(filter_message)
    server.serve_forever()


def filter_message(session, lines):
    # Modify email lines here if necessary
    return lines


if __name__ == '__main__':
    start()
```

Adding it to `smtpd.conf` is also largely straightforward:

```ini
# Define dmarc2html filter
filter dmarc2html proc-exec "/opt/dmarc2html/filter.py"
# Chain all filters to be applied to incoming mail
filter incoming_chain chain {filter1, filter2, dmarc2html}
# Apply filter
listen on eth0 tls pki default filter incoming_chain
```

## Restricting the filter to a particular recipient

OpenSMTPD consults filters immediately during all the steps of email delivery. This means that limiting a filter to a specific recipient isn’t possible, initially the email server simply doesn’t know who the recipient will be.

But I only want to handle emails directed at the designated recipient of DMARC aggregate reports. My initial approach was to parse the email and to check the `To` header. This approach is rather wasteful however, parsing all incoming emails trying to find the few “right” ones. More importantly: a `To` header doesn’t have to be present, and it doesn’t have to designate the real recipient.

What we really want to check is the email address given in the `RCPT TO` SMTP command. This is possible by registering a handler for the `tx-rcpt` event. We can store the recipient in the session context for later use:

```python
server.track_context()
server.register_handler('report', 'tx-rcpt', save_rcpt)

…

def save_rcpt(session, message_id, result, address):
    if result != 'ok':
        return
    session['rcpt'] = address
```

Now that we have the recipient in the session context, we can check it in the message filter:

```python
def filter_message(session, lines):
    if not session['rcpt'].startswith('dmarc@'):
        return lines

    # This is a DMARC aggregate report, modify it here
    return lines
```

Note that checking whether `rcpt` entry exists is unnecessary here, OpenSMTPD won’t allow email delivery attempts without a preceding `RCPT TO` command.

## Processing the email

For email processing, Python has the very useful but also rather underdocumented [email package](https://docs.python.org/3/library/email.html). E.g. here is how you parse an email:

```python
import email
import email.policy

…

parsed = email.message_from_string('\n'.join(lines), policy=email.policy.default)
```

Counterintuitively, the `default` policy isn’t the one the parser uses by default. The `compat32` policy used by default will give you a `Message` instance which is only good for accessing email headers. The `default` policy on the other hand will give you an `EmailMessage` instance which lets you inspect the email body.

Normally, email attachments come as part of a multipart email. However, with aggregate reports sent by Gmail for example the entire email is an attachment. As `EmailMessage` behavior changes considerably depending on whether it wraps a multipart email, I decided to convert such attachment emails to multipart emails before continuing:

```python
if not parsed.is_multipart() and parsed.get_filename() is not None:
    parsed.make_mixed()
```

Now each valid aggregate report should have one attachment:

```python
attachments = [*parsed.iter_attachments()]
if len(attachments) != 1:
    raise Exception('Expected one attachment, got {}'.format(len(attachments)))
```

And we can get the HTML code corresponding to this attachment:

```python
filename = attachment[0].get_filename()
contents = attachment[0].get_payload(decode=True)
html = dmark2html.process_report(filename, contents)
```

We need this HTML code wrapped in a `MIMEPart` object:

```python
import email.message

…

html_part = email.message.MIMEPart(policy=email.policy.default)
html_part.set_content(html, subtype='html', charset='utf-8')
```

Now we can replace the body of our email and return the new lines from the filter:

```python
parsed.set_payload([html_part, attachments[0]])
return parsed.as_string().strip().split('\n')
```

## The result

The complete code is [available on GitHub](https://github.com/palant/dmarc2html/). While not considerably more complicated than what has been presented here, it makes a few additional considerations. In particular, one wouldn’t want to drop the email in case of some exception. So the handler makes certain to log exceptions while returning the unchanged lines in that case.

It also makes the DMARC recipient account configurable via a command line parameter. So when emails are received by this account, what I see now is no longer a blank email with an attachment but this:

{{< img src="dmarc_report.png" width="793" alt="Screenshot of an email with the subject “Report domain: palant.de Submitter: google.com Report-ID: 2768102006723386361” Email text says: DMARC aggregate report from google.com covering the time span from 2023-03-05 00:00:00 UTC to 2023-03-05 23:59:59 UTC: Source IP: 91.216.248.201 (a.mail-out.lima-city.de). Count: 2. Disposition: none. DKIM: pass 👁. SPF: pass 👁" />}}

The eye symbol indicates that additional information is available by hovering the table cell. This way the output isn’t cluttered with unnecessary details.