# 使用Docker Mailserver搭建邮件服务器并用Roundcube Webmail管理



## 配置Docker Mailserver

下载配置文件，要对docker-compose.yml稍作修改，添加邮件中继。

```bash
DMS_GITHUB_URL='https://raw.githubusercontent.com/docker-mailserver/docker-mailserver/master'
wget "${DMS_GITHUB_URL}/docker-compose.yml"
wget "${DMS_GITHUB_URL}/mailserver.env"
# wget "${DMS_GITHUB_URL}/setup.sh"
# chmod a+x ./setup.sh
```

完整配置文件如下，其中<>内的信息自行替换，mailserver.env为之前下载的原始文件，没有修改。

如果是裸域安装，就在hostname内直接填写`<domain>.<tld>`。

手动配置ssl，调用服务器配置的ssl文件，其他方法参考官方文档。

为避免垃圾邮件泛滥，大部分服务器提供商通常会封锁25端口出站流量，不能直接发邮件，因此配置Mailgun作为邮件中继发件。

第26行和第29、30行证书位置自行替换为对应位置。

自签名证书直接套用此配置会报错，需另外配置，可去官方文档自行查询。

```yaml
version: "3"
services:
  mailserver:
    image: mailserver/docker-mailserver:11.3.1
    container_name: mailserver
    # If the FQDN for your mail-server is only two labels (eg: example.com),
    # you can assign this entirely to `hostname` and remove `domainname`.
    hostname: <hostname>
    domainname: <domain>.<tld>
    env_file: mailserver.env
    # More information about the mail-server ports:
    # https://docker-mailserver.github.io/docker-mailserver/edge/config/security/understanding-the-ports/
    # To avoid conflicts with yaml base-60 float, DO NOT remove the quotation marks.
    ports:
      - "25:25" # SMTP  (explicit TLS => STARTTLS)
      - "143:143" # IMAP4 (explicit TLS => STARTTLS)
      - "465:465" # ESMTP (implicit TLS)
      - "587:587" # ESMTP (explicit TLS => STARTTLS)
      - "993:993" # IMAP4 (implicit TLS)
    volumes:
      - ./mail-data/:/var/mail/
      - ./mail-state/:/var/mail-state/
      - ./mail-logs/:/var/log/mail/
      - ./config/:/tmp/docker-mailserver/
      - /etc/localtime:/etc/localtime:ro
      - /etc/nginx/ssl/:/etc/letsencrypt/:ro
    environment:
      - SSL_TYPE=manual
      - SSL_CERT_PATH=/etc/letsencrypt/certs/<cert>.cer
      - SSL_KEY_PATH=/etc/letsencrypt/private/<private>.key
      - ENABLE_SPAMASSASSIN=1
      - ENABLE_FAIL2BAN=1
      - POSTFIX_MESSAGE_SIZE_LIMIT=20971520 # 20M
      - UPDATE_CHECK_INTERVAL=7d
      - RELAY_HOST=smtp.mailgun.org
      - RELAY_PORT=587
      - RELAY_USER=<user>
      - RELAY_PASSWORD=<password>
    restart: always
    stop_grace_period: 1m
    healthcheck:
      test: "ss --listening --tcp | grep -P 'LISTEN.+:smtp' || exit 1"
      timeout: 3s
      retries: 0
```

15-mailboxes.conf为自定义邮箱文件夹配置，主要添加了存档文件夹，放在”./config/dovecot”下，也可以自行修改。

```text
##
## Mailbox definitions
##

# Each mailbox is specified in a separate mailbox section. The section name
# specifies the mailbox name. If it has spaces, you can put the name
# "in quotes". These sections can contain the following mailbox settings:
#
# auto:
#   Indicates whether the mailbox with this name is automatically created
#   implicitly when it is first accessed. The user can also be automatically
#   subscribed to the mailbox after creation. The following values are
#   defined for this setting:
#
#     no        - Never created automatically.
#     create    - Automatically created, but no automatic subscription.
#     subscribe - Automatically created and subscribed.
#
# special_use:
#   A space-separated list of SPECIAL-USE flags (RFC 6154) to use for the
#   mailbox. There are no validity checks, so you could specify anything
#   you want in here, but it's not a good idea to use flags other than the
#   standard ones specified in the RFC:
#
#     \All       - This (virtual) mailbox presents all messages in the
#                  user's message store.
#     \Archive   - This mailbox is used to archive messages.
#     \Drafts    - This mailbox is used to hold draft messages.
#     \Flagged   - This (virtual) mailbox presents all messages in the
#                  user's message store marked with the IMAP \Flagged flag.
#     \Important - This (virtual) mailbox presents all messages in the
#                  user's message store deemed important to user.
#     \Junk      - This mailbox is where messages deemed to be junk mail
#                  are held.
#     \Sent      - This mailbox is used to hold copies of messages that
#                  have been sent.
#     \Trash     - This mailbox is used to hold messages that have been
#                  deleted.
#
# comment:
#   Defines a default comment or note associated with the mailbox. This
#   value is accessible through the IMAP METADATA mailbox entries
#   "/shared/comment" and "/private/comment". Users with sufficient
#   privileges can override the default value for entries with a custom
#   value.

# NOTE: Assumes "namespace inbox" has been defined in 10-mail.conf.
namespace inbox {
  # These mailboxes are widely used and could perhaps be created automatically:
  mailbox Drafts {
    auto = subscribe
    special_use = \Drafts
  }
  mailbox Junk {
    auto = subscribe
    special_use = \Junk
  }
  mailbox Trash {
    auto = subscribe
    special_use = \Trash
  }

  # For \Sent mailboxes there are two widely used names. We'll mark both of
  # them as \Sent. User typically deletes one of them if duplicates are created.
  mailbox Sent {
    auto = subscribe
    special_use = \Sent
  }

  #mailbox "Sent Messages" {
  #  special_use = \Sent
  #}

  mailbox Archive {
   auto = subscribe
   special_use = \Archive
  }

  # If you have a virtual "All messages" mailbox:
  #mailbox virtual/All {
  #  special_use = \All
  #  comment = All my messages
  #}

  # If you have a virtual "Flagged" mailbox:
  #mailbox virtual/Flagged {
  #  special_use = \Flagged
  #  comment = All my flagged messages
  #}

  # If you have a virtual "Important" mailbox:
  #mailbox virtual/Important {
  #  special_use = \Important
  #  comment = All my important messages
  #}
}
```

## 配置DNS和防火墙

### DNS配置

裸域安装是没有必要的，无论是不是裸域，邮件后缀`@<domain>.<tld>`都能够收到。

但是裸域安装要在”./config”文件夹下添加配置文件”postfix-main.cf”，加入以下内容，官方文档有写。

```text
mydestination = localhost.$mydomain, localhost
```

关键在于MX解析的配置。

以邮件服务器域名为`mail.<domain>.<tld>`为例，配置A解析mail到服务器ip地址，配置MX解析  `<domain>.<tld>`到`mail.<domain>.<tld>`，优先级设置为10，这样就保证了发往`@<domain>.<tld>`的邮件能够到达。

生成DKIM等配置由于采用了邮件中继，因此直接使用邮件中继提供的内容即可，无需自行配置。以Mailgun为例，位于Sending > Domain settings > DNS records处。添加这些DNS记录是较为必要的，保证了邮件的送达率。

注意，MX记录不要按照Mailgun给出的配置，其配置将收件指向Mailgun。

### 防火墙

防火墙方面，25端口需要开放才能收到邮件。143端口为未加密IMAP，可以不开放。465端口和587端口为加密SMTP端口，建议开放。993端口为加密IMAP端口，建议开放。

## 用户管理

这里列出了创建用户，添加别名，设置邮件配额的基本方法。

```bash
# 添加用户
docker exec -it mailserver setup email add <user>@<domain>.<tld> <password>
# 创建用户别名
docker exec -it mailserver setup alias add <alias>@<domain>.<tld> <user>@<domain>.<tld>
# 设置邮件配额
docker exec -it mailserver setup quota set <user>@<domain>.<tld> 2G
```

更详细的内容自行查看官方文档。

## 小插曲

使用一段时间后我查看log，发现有个ip一直在尝试破解登录我的邮箱，尝试了不同的用户名。遇到这样的情况，我们可以使用内置的fail2ban工具手动ban掉他的ip。

启用/重启fail2ban工具。

```bash
docker exec -it mailserver supervisorctl restart fail2ban
```

Ban掉指定ip地址。

```bash
docker exec -it mailserver setup fail2ban ban 45.128.234.165
# 取消禁令
# docker exec -it mailserver setup fail2ban unban 45.128.234.165
```

## 安装Roundcube Webmail

roundcube是一个网络邮件客户端，可以通过如下配置文件安装。

假定Docker Mailserver服务器的域名为`mail.<domain>.<tld>`。

在环境中指定IMAP和SMTP端口，设置附件最大文件大小。也可采用mysql等数据库作为后端，但不是特别必要。

```yaml
version: '3'

services:
  roundcube:
    image: roundcube/roundcubemail:1.6.x-apache
    container_name: roundcube
    restart: unless-stopped
    volumes:
      - ./data:/var/www/html
      - ./db/sqlite:/var/roundcube/db
      - ./config:/var/roundcube/config
    ports:
      - <port>:80
    environment:
      - ROUNDCUBEMAIL_DB_TYPE=sqlite
      - ROUNDCUBEMAIL_SKIN=elastic
      - ROUNDCUBEMAIL_DEFAULT_HOST=ssl://mail.<domain>.<tld>
      - ROUNDCUBEMAIL_DEFAULT_PORT=993
      - ROUNDCUBEMAIL_SMTP_SERVER=ssl://mail.<domain>.<tld>
      - ROUNDCUBEMAIL_SMTP_PORT=465
      - ROUNDCUBEMAIL_UPLOAD_MAX_FILESIZE=20M
```

配置Apache2或Nginx等http服务器通过SSL反向代理到端口是更安全的。这里给出一个Nginx的例子，按照实际情况修改。

```text
server {
    listen 80;
    listen [::]:80;
    server_name <host>.<domain>.<tld>;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name <host>.<domain>.<tld>;

    client_max_body_size 20m;

    ssl_certificate      /etc/nginx/ssl/certs/*.<domain>.<tld>-fullchain.cer;
    ssl_certificate_key  /etc/nginx/ssl/private/*.<domain>.<tld>.key;

    ssl_session_cache    shared:SSL:1m;
    ssl_session_timeout  5m;

    ssl_ciphers  HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers  on;

    location / {
        proxy_pass http://127.0.0.1:<port>/;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_cache_bypass $http_upgrade;

        proxy_set_header Host  $http_host;
        proxy_set_header X-Real-IP  $remote_addr;
        proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto  $scheme;
    }

    access_log  /var/log/nginx/webmail.access.log;
    error_log   /var/log/nginx/webmail.error.log;
}
```

## 一些链接

docker-mailserver官方文档：<https://docker-mailserver.github.io/docker-mailserver/v10.0/introduction/>

docker-mailserver托管地址：<https://github.com/docker-mailserver/docker-mailserver>

roundcube-docker托管地址：<https://github.com/roundcube/roundcubemail-docker>

