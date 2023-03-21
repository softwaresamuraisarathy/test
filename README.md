import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

# Set up the SMTP server connection
smtp_server = "smtp.gmail.com"
smtp_port = 587
smtp_username = "your-email@gmail.com"
smtp_password = "your-email-password"

# Create the message object
msg = MIMEMultipart('alternative')
msg['From'] = "your-email@gmail.com"
msg['To'] = "recipient-email@example.com"
msg['Subject'] = "Test email with plain text and HTML content"

# Create the plain text message body
text = "Hello, this is a test email with plain text content."

# Create the HTML message body
html = """\
<html>
  <body>
    <p>Hello,<br>
       This is a test email with <b>HTML</b> content.
    </p>
  </body>
</html>
"""

# Attach both the plain text and HTML versions of the message body
part1 = MIMEText(text, 'plain')
part2 = MIMEText(html, 'html')
msg.attach(part1)
msg.attach(part2)

# Send the message via the SMTP server
with smtplib.SMTP(smtp_server, smtp_port) as server:
    server.starttls()
    server.login(smtp_username, smtp_password)
    server.sendmail(smtp_username, msg['To'], msg.as_string())
