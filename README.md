# Correction Program for Japanese Localization Support in Mailman 3

## Description
The Mailman 3 system has many parts that have not been translated into Japanese on the web interface. Additionally, there is a critical issue where sending Japanese emails to the mailing list results in garbled text or even loss of emails. This is a corrective program to address these issues.

## Target Software
- Mailman Core v3.3.9
- Postorius v1.3.10
- Hyperkitty v1.3.8
- Mailmanclient v3.3.5
- Hyperkitty Mailman plugin v1.2.1
- Django-mailman3 v1.3.11

Verification has been conducted on AlmaLinux 9.3.

## Application Procedure
In this example, let's assume that Mailman 3 is installed in the directory /opt/mailman/.

### mailman3-japanese-web.patch
Apply the downloaded program.
```
# cd /opt/mailman/venv/lib/python3.X/
# patch -p0 < mailman3-japanese-web.patch
# chown -R mailman:mailman /opt/mailman/venv/lib/python3.X/
```

Recreate the translation file with the user 'mailman'.
```
# su mailman
$ cd
$ mailman-web compilemessages
$ msgfmt /opt/mailman/venv/lib/python3.X/site-packages/mailman/messages/ja/LC_MESSAGES/mailman.po -o /opt/mailman/venv/lib/python3.X/site-packages/mailman/messages/ja/LC_MESSAGES/mailman.mo
```

### mailman3-japanese-enhancement.patch
Install python_iconv.
```
# su mailman
$ pip install python_iconv
```

The Japanese email correction program uses libiconv. Therefore, let's start by installing libiconv.
Please obtain libiconv from the following website: https://github.com/designet-inc-oss/libiconv
```
$ tar xzf libiconv-X.XX.tar.gz
$ patch -p0 < libiconv-X.XX.-ja-4.patch
$ cd libiconv-X.XX
$ ./configure
$ make
# make install
```

Modify the unit file.
```
# vi /etc/default/mailman3
LD_LIBRARY_PATH=/usr/local/lib
LD_PRELOAD=/usr/local/lib/preloadable_libiconv.so
CHARSET_ALIAS="Shift_JIS=CP932:EUC-JP=EUC-JP-MS:ISO-2022-JP=ISO-2022-JP-MS"
```
```
# vi /etc/systemd/system/mailman3.service
[Unit]
Description=GNU Mailing List Manager
After=syslog.target network.target postgresql.service

[Service]
Type=forking
PIDFile=/opt/mailman/mm/var/master.pid
User=mailman
Group=mailman
EnvironmentFile=/etc/default/mailman3         <- Add the following line.
ExecStart=/opt/mailman/venv/bin/mailman start
ExecReload=/opt/mailman/venv/bin/mailman restart
ExecStop=/opt/mailman/venv/bin/mailman stop

[Install]
WantedBy=multi-user.target
```
```
# vi /etc/systemd/system/mailmanweb.service
[Unit]
Description=GNU Mailman Web UI
After=syslog.target network.target postgresql.service mailman3.service

[Service]
Environment="PYTHONPATH=/etc/mailman3/"
EnvironmentFile=/etc/default/mailman3         <- Add the following line.
User=mailman
Group=mailman
ExecStart=/opt/mailman/venv/bin/uwsgi --ini /etc/mailman3/uwsgi.ini
KillSignal=SIGINT

[Install]
WantedBy=multi-user.target
```

Apply the patch.
```
# cd /opt/mailman/venv/lib/python3.X/
# patch -p0 < mailman3-japanese-enhancement.patch
```

Reflect the configuration changes.
```
# systemctl daemon-reload
# systemctl restart mailman3
# systemctl restart mailmanweb
```
