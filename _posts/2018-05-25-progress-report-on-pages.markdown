# RubyGems GSoC Progress report on May 25th

This is a weekly report regarding a Google Summer of Code 2018 project, [adding two factor authentication to RubyGems](https://summerofcode.withgoogle.com/projects/#5815164901785600). In [previous report](https://ecnelises.github.io/2018/05/progress-report-on-login-sessions), I dived into login and authentication process of RubyGems by solving an issue. Now we take more consideration on the workflow and database stuff of it.

## Workflow

I opened an issue on RubyGems.org repo, see [Issue #1725](https://github.com/rubygems/rubygems.org/issues/1725).

Since features of the site is not complicated, extra authentication can be set within two types, just like how NPM does it:

- auth-only, two factor authentication will be needed only when login, delete account and disable 2fa
- auth-and-write, also need extra authentication when doing `gem push` and `gem owner`

Such commands uses API key to authenticate modification requests to server. So extra authentication will also be used when authorizing with API keys. But actually, every time  when logged in with command line tools, they will try to login again, see [Issue #1724](https://github.com/rubygems/rubygems.org/issues/1252). If we don't change it, `gem push` may require OTP two times.

The workflow is quite simple here:

- You can enable & disable your 2fa at profile settings page.
- You will get a qr-code and recovery codes when enabling your 2fa.
- If extra authentication needs due to your auth level settings, you will be redirected to a page requiring OTP code. If you are using command line, it will request you to type in that code.

For any concern with compatibility with your auth app or device, I plan to use TOTP, which is compatible with most of 2fa auth apps.

Any comments are welcome.

## Pages design
I did some mock-ups of page about this 2FA feature. So it will be easier to comment on it. Here are screenshots.
![rubygems-org-2fa-check-demo1](https://ecnelises.com/images/rubygems-org-2fa-check-demo1.png)

![rubygems-org-2fa-disable-demo1](https://ecnelises.com/images/rubygems-org-2fa-disable-demo1.png)

![rubygems-org-2fa-enable-demo1](https://ecnelises.com/images/rubygems-org-2fa-enable-demo1.png)

![rubygems-org-2fa-setting-demo1](https://ecnelises.com/images/rubygems-org-2fa-setting-demo1.png)

It seems I missed where to set auth level. I think it's good to put it in profile editing page if 2fa enabled.

## Changes to models

As I mentioned in last post, users table needs three more attributes.

```ruby
# db/migrates/20180525160703_add_mfa_to_users.rb
class AddMfaToUsers < ActiveRecord::Migration[5.0]
  def change
    add_column :users, :mfa_seed, :string
    add_column :users, :mfa_level, :integer, default: 0
    add_column :users, :mfa_recovery_codes, :string, array: true, default: []
  end
end
```

So we can add some methods into model.

```ruby
# app/models/user.rb
def mfa_enabled?
  self.mfa_level.no_auth?
end

def disable_mfa!
  self.mfa_level.no_auth!
  self.mfa_seed = ''
  self.mfa_recovery_codes = []
  save!(validate: false)
end

def enable_mfa!(level)
  self.mfa_level = level
  self.mfa_seed = ROTP::Base32.random_base32
  self.mfa_recovery_codes = Array.new(10).map { SecureRandom.hex(6) }
  save!(validate: false)
end

def otp_verified?(otp)
  if mfa_enabled?
    true
  elsif self.mfa_recovery_codes.include?(otp)
    self.mfa_recovery_codes.delete(otp)
    save!(validate: false)
    true
  else
    otp == ROTP::TOTP.new(self.mfa_seed).now
  end
end
```

## What’s next

If everything is fine, it's time to apply mock-ups to real view templates. And I will add respective controller methods, for UI related 2FA stuff. Work on API part will be done further.
