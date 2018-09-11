---
title: Tunneling pop3/smtp to gmail SSL pop3s/smtps using xinetd on Linux
date: 2010-06-29T06:16:25+00:00
tags: [ "technical", "linux", "gmail", "xinetd", "stunnel" ]
draft: false
---

My Brother MFC-9840cdw multi-functional printer/scanner/coper/fax can scan to email. However gmail requires SSL, which the printer is not able to support. Using xinetd and openssl, my Linux machine is able to proxy local pop3 requests to gmail's SSL pop3s service, and local smtp requests to gmail's SSL smtps service.

Here are the basics on what you need. These ideas are not original.

Tunneling smtp to smtps

{{<highlight go "">}}
# xinetd: tunnel local smtp to gmail smtps

service smtp
{
  disable = no
  socket_type = stream
  wait = no
  user = root
  server = /etc/xinetd.d/extra/gmailsmtpstunnel.sh
  only_from = 0.0.0.0
}

#####################################################################
# Instructions
#
#
# 1) Copy this entire file to /etc/xinetd.d/gmailsmtpstunnel
#
# 2) Create the following script in:
#      /etc/xinetd.d/extra/gmailsmtpstunnel.sh
# --------------------------------------
# #!/bin/sh
# /usr/bin/openssl s_client -quiet -connect smtp.gmail.com:465 2>/dev/null
# --------------------------------------
# 3) Change the permissions
#     chmod 755 /etc/xinet.d/extra/gmailsmtpstunnel.sh
# 4) Restart xinetd
#      /etc/init.d/xinetd restart
# 5) Test the tunnel
#      telnet localhost 25
# 6) Point your mail smtp server to here

{{</highlight>}}

Tunneling pop3 to pop3s

{{<highlight go "">}}
# xinetd: tunnel local pop3 to gmail pop3s

service pop3
{
  disable = no
  socket_type = stream
  wait = no
  user = root
  server = /etc/xinetd.d/extra/gmailpop3stunnel.sh
  only_from = 0.0.0.0
}

#####################################################################
# Instructions
#
#
# 1) Copy this entire file to /etc/xinetd.d/gmailpop3stunnel
#
# 2) Create the following script in:
#      /etc/xinetd.d/extra/gmailpop3stunnel.sh
# --------------------------------------
# #!/bin/sh
# /usr/bin/openssl s_client -quiet -connect pop.gmail.com:995 2>/dev/null
# --------------------------------------
# 3) Change the permissions
#     chmod 755 /etc/xinet.d/extra/gmailpop3stunnel.sh
# 4) Restart xinetd
#      /etc/init.d/xinetd restart
# 5) Test the tunnel
#      telnet localhost 110
# 6) Point your mail smtp server to here
{{</highlight>}}