+++
title = "Configuring Magit Forge with the FreeDesktop Secrets API"
slug = "configuring-forge-with-freedesktop-secrets-api"
date = "2022-05-27"
category = "emacs"

[extra]
author = "Bob Gregory"

[taxonomies]
tags = ["emacs", "linux"]
+++

It took me an hour or so to work this one out this morning. If you're running Magit Forge under Emacs and you want to store a personal access token on a recent Linux desktop environment, then you might want to use the Freedesktop secrets api.
<!-- more -->

The documentation is a little bit sparse, but I set this up using [Gnome Keyring](https://wiki.archlinux.org/title/GNOME/Keyring) and [secret-tool](https://man.archlinux.org/man/secret-tool.1.en).

First you'll need to generate your personal access token with `user`, `repo`, and `org:read` permissions per [the documentation](https://Magit.vc/manual/ghub/Creating-a-Token.html). That'll give you a token, eg `ghp_TwEeTTweEtSaYsTheBirDyASItfliesPAst1`.

You need to store this in your keyring, like so:

```console
bob@bobs-spangly-carbon ~> secret-tool store \
  --label='Forge Github token' \
  host api.github.com \
  user $USERNAME^forge
```

In my case, my github username is bobthemighty, so I provide the user `bobthemighty^forge`. The `^forge` at the end is used by Magit as a discriminator so it can find the right set of credentials.
That'll prompt you for a password, and you'll need to enter the token from before.

```console
bob@bobs-spangly-carbon ~> secret-tool store \
  --label='Forge Github token' \
  host api.github.com \
  user $USERNAME^forge

Password: ghp_TwEeTTweEtSaYsTheBirDyASItfliesPAst1
```

Lastly, you need to configure Magit to read secrets from the secrets api.

```lisp
(setq auth-sources '(default
                     "secrets:login"
                     ))
```

This tells Magit to read from the `login` collection in the secret store, which is the default collection created by Gnome Keyring. Reload your Emacs config and you should be good to go.
