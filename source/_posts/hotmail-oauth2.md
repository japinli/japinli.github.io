---
title: Hotmail 邮箱 OAuth2 认证
tags:
  - OAtuth2
  - mbsync
  - msmtp
categories: 杂项
date: 2024-10-13 14:47:09
---


自 2024 年 09 月 26 日起，微软的 Hotmail 邮箱将不再支持第三方电子邮件客户端使用用户名和密码的登录方式，从而导致我的邮件客户端不能正常使用，必须使用 OAuth2 的认证方式才能访问。

{% asset_img microsoft-security-limit.jpg %}

<!--more-->

## 解决办法

[Simon Dobson 的文章](https://simondobson.org/2024/02/03/getting-email/)提到了如何为 [mbsync](https://github.com/gburd/isync) 添加 OAuth2 认证。但是，由于我的 Hotmail 邮箱是个人账号，因此按照文章中给出的方法，我仍然无法获取到访问邮箱的 token。文章中提到的 `client_id` 和 `client_secret` 来自于 [Configure Mutt to work with OAuth 2.0](https://www.dcs.gla.ac.uk/~jacobd/posts/2022/03/configure-mutt-to-work-with-oauth-20/)，然而，Simon Dobson 说了他不知道这两个值来自于哪儿，并且他指出这与 Thunderbird 源码中的值不同。因此，我也跑去 Thunderbird 中寻找线索（Simon Dobson 文章中给出的 Thunderbird 链接目前已经失效，正确的链接在[这儿](https://hg.mozilla.org/comm-central/file/tip/mailnews/base/src/OAuth2Providers.sys.mjs)）。

我发现在 [Thunderbird](https://hg.mozilla.org/comm-central/file/tip/mailnews/base/src/OAuth2Providers.sys.mjs#l139) 的源码中都只有 `clientId` 而没有所谓的 `client_secret`，如下所示：

```
  [
    "login.microsoftonline.com",
    {
      clientId: "9e5f94bc-e8a4-4e73-b8be-63364c29d753", // Application (client) ID
      // https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-v2-protocols#endpoints
      authorizationEndpoint:
        "https://login.microsoftonline.com/common/oauth2/v2.0/authorize",
      tokenEndpoint:
        "https://login.microsoftonline.com/common/oauth2/v2.0/token",
      redirectionEndpoint: "https://localhost",
    },
  ],
```

随后我又在 [OAuth2.sys.mjs](https://hg.mozilla.org/comm-central/file/tip/mailnews/base/src/OAuth2.sys.mjs#l302) 中发现了如下内容：

```
    data.append("client_id", this.clientId);
    if (this.consumerSecret !== null) {
      // Section 2.3.1. of RFC 6749 states that empty secrets MAY be omitted
      // by the client. This OAuth implementation delegates this decision to
      // the caller: If the secret is null, it will be omitted.
      data.append("client_secret", this.consumerSecret);
    }
```
这是否意味着 `client_secret` 的值可以为空呢？抱着试一试的心态尝试了一下，发现还真是这样。

## 完整步骤

1. 使用 `gpg --gen-key` 生成密钥，后续将会使用该密钥加密存储 token 信息在。这个过程中会要求输入名称和邮件地址，例如，下面是我生成的示例：

    ```
    pub   ed25519 2024-10-12 [SC] [expires: 2026-10-12]
          CD46A95EB467DFCED20DDC871AD1DA436506096C
    uid                      Hotmail Token Encryption <japinli@hotmail.com>
    sub   cv25519 2024-10-12 [E] [expires: 2026-10-12]
    ```

2. mbsync 没有提供内置的 OAuth2 处理，需要借助 `mutt_oauth2.py` 来获取 token。在这之前，我们需要通过 gpg 创建一个密钥来对 token 进行加密存储。

    ```
    $ curl -o mutt_oauth2.py https://gitlab.com/muttmua/mutt/-/raw/master/contrib/mutt_oauth2.py
    $ chmod u+x mutt_oauth2.py
    ```

3. 修改 `mutt_oauth2.py`，将 `YOUR_GPG_IDENTITY` 替换为步骤 1 中的密钥标识，例如：`Hotmail Token Encryption <japinli@hotmail.com>`，同时，将 `microsoft` 下的 `client_id` 的值设置为 `9e5f94bc-e8a4-4e73-b8be-63364c29d753`。

4. 然后以“授权”模式运行：`./mutt_oauth2.py -t japinli@hotmail.com.token --authorize`
   * OAuth2 registration: microsoft
   * Preferred OAuth2 flow: authcode
   * Account e-mail address: japinli@hotmail.com

   输入完上述内容之后，将会返回一个链接，使用浏览器打开该链接，在链接后有个 `code` 的参数，将其值复制到提示处，按下 Enter 键，就能获取到 token 信息。

5. 使用 `./mutt_oauth2.py -t japinli@hotmail.com.token` 可以获取 token 并自动刷新。
   我当时遇到了如下问题：
    ```
    Difficulty decrypting token file.
    Is your decryption agent primed for non-interactive usage,
    or an appropriate environment variable such as GPG_TTY set to allow interactive agent usage from inside a pipe?
    ```
   通过设置 `GPG_TTY` 可以解决：`export GPG_TTY=$(tty)`。 然而，更多的时候是想要不输入口令就能直接获取 token，这样 mbsync 就可以自动拉取，而不需要用户输入口令。这个可以通过修改 `mutt_oauth2.py` 中的 `DECRYPTION_PIPE` 来解决。
    ```
    DECRYPTION_PIPE = ['gpg', '--pinentry-mode=loopback', '--batch', '--yes',
                       '--passphrase-file', '/path/to/passphrase-file',
                       '--decrypt']
    ```

6. 修改 mbsync 的认证方式：

    ```
    AuthMechs OATHOU2
    PassCmd "/path/to/mutt_oauth2.py -t /path/to/japinli@hotmail.com.token"
    ```

经过上述 6 个步骤之后，mbsync 可以正常获取 Hotmail 中的邮件了，但是发送邮件还不能正常工作，这是由于我们还没有为 smtp 配置 OAuth2 认证。

我尝试用这篇文章中的方式为 mu4e 的 `smtpmail-sent-it` 配置 OAuth2 认证，虽然在登录的时候可以正确获取到 token 信息，但是始终不能正常发送邮件。

最后，我采用了 [msmtp](https://github.com/marlam/msmtp) 来发送邮件。配置信息如下所示：

```
# Set default values for all following accounts.
defaults
auth    on
tls     on
tls_starttls  on
tls_trust_file  /etc/ssl/certs/ca-certificates.crt
logfile /tmp/msmtp.log

account hotmail
host  smtp.office365.com
port  587
auth  xoauth2
user  japinli@hotmail.com
from  japinli@hotmail.com
passwordeval "/path/to/mutt_oauth2.py /path/to/token"

account default : hotmail
```

配置完成之后，我测试 msmtp 时 `passwordeval` 报 `/path/to/mutt_oauth2.py /path/to/token` 不能正确执行，这是由于 [AppArmor][https://ubuntu.com/server/docs/apparmor] 对 msmtp 进行了限制，禁用之后就可以正常使用了。

```
$ sudo ln -s /etc/apparmor.d/usr.bin.msmtp /etc/apparmor.d/disable/
$ apparmor_parser -R /etc/apparmor.d/disable/usr.bin.msmtp
```

最后，我们需要修改 mu4e 中关于发送邮件的部分，如下所示：

```emacs-lisp
(setq sendmail-program "/usr/bin/msmtp"
      send-mail-function 'smtpmail-send-it
      message-sendmail-f-is-evil t
      message-sendmail-extra-arguments '("--read-envelope-from")
      message-send-mail-function 'message-send-mail-with-sendmail)
```

## 参考

[1] https://wiki.archlinux.org/title/Isync#Using_XOAUTH2
[2] https://simondobson.org/2024/02/03/getting-email/
[3] https://www.dcs.gla.ac.uk/~jacobd/posts/2022/03/configure-mutt-to-work-with-oauth-20/
[4] https://hg.mozilla.org/comm-central/file/tip/mailnews/base/src/OAuth2Providers.sys.mjs
[5] https://www.macs.hw.ac.uk/~rs46/posts/2022-01-11-mu4e-oauth.html
[6] https://tushartyagi.com/blog/configure-mu4e-and-msmtp/
[7] https://linuxconfig.org/how-to-disable-apparmor-on-ubuntu-20-04-focal-fossa-linux
