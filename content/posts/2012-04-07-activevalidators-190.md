---
title: ActiveValidators 1.9.0
date: 2012-04-07
slug: "activevalidators-190"
tags:
  - ruby
  - open-source
---

ActiveValidators 1.9.0 is out!

```ruby
gem install activevalidators
```

Read on for the full - yet concise - changelog.

<!--more-->

From its first versions, ActiveValidators has been designed to rely on the
`mail` gem. The `mail` gem is at the foundation of `action_mailer`, and doing
so reduced the chances of having emails validated by ActiveValidators and
rejected by `action_mailer` or MTAs.

The EmailValidator can validate emails in the form `user@domain.tld`,
but `mail` accepts any RFC-valid email address (1), like `"User" <user@domain.tld>`.
Although that kind of format is correct, most apps want to only keep the `user@domain.tld` part
and leave the rest.

Since [this commit](https://github.com/franckverrot/activevalidators/commit/d77d9105e5831bfac16cadbeaf9dc055d5e941ff), the email validator now accepts an option
to constraint a bit more the email validation:

```ruby
validates :field_to_validate, :email => { :strict => true/false }
```

By default, `strict` is set to `false` and the `mail` gem validation will apply.
When `strict` is set to `true`, only emails in the form `user@domain.tld` will be valid.

I'm wondering it would make sense to switch the default to `true` (I tend to prefer sensible defaults).
Don't hesitate to give an opinion on that and thanks for using ActiveValidators.


1. [RFC 5321](http://tools.ietf.org/html/rfc5321), [RFC 5322](http://tools.ietf.org/html/rfc5322) and [RFC 5396](http://tools.ietf.org/html/rfc3696).
