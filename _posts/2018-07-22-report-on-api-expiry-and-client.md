# RubyGems Weekly Report on Jul 22nd

This is a weekly report on my newest progress of the GSoC (Google Summer of Code) project on _Adding Multi-factor Authentication to RubyGems_. In this week, I added expiry of QR-codes, filter of OTP on API authentication, and OTP support to `gem` client. Some of them are not committed or included in a pull request yet.

## Expiry of QR-code

Since QR-code (or the key, actually) of multi-factor authentication is stored in session and all the extra security relies on it, storing them in session forever (think of a case when you opened page with QR-code and close it without sending request to enable it) is not a good way. Here I add a separate expiry in session.

```ruby
# app/models/user.rb
def verify_and_enable_mfa!(seed, level, otp, expiry)
  if expiry < Time.now.utc
    errors.add(:base, I18n.t('multifactor_auths.create.qrcode_expired'))
  elsif verify_digit_otp(seed, otp)
    enable_mfa!(seed, level)
  else
    errors.add(:base, I18n.t('multifactor_auths.incorrect_otp'))
  end
end
```

And expiry interval is set in application constants.

```ruby
# config/application.rb
MFA_KEY_EXPIRY = 30.minutes
```

That works well, except that some people may think the period is too long.

## Improvements on API

My last post did not mention things about authentication on API part. Actually it's simpler. First, the way is:

- If an action requires multi-factor authentication and `mfa_login_and_write` option is set for current user, OTP is required.
- OTP is fetched through `OTP` field in HTTP header.
- If OTP is verified, everything goes on.
- Otherwise, a `401 Unauthorized` will be returned.

So let's write a filter first.

```ruby
# app/controllers/application_controller.rb
def verify_with_otp
  return unless @api_user.mfa_login_and_write?
  otp = request.headers["HTTP_OTP"] || ''
  return if @api_user.otp_verified?(otp)
  render plain: t(:please_send_correct_otp), status: :unauthorized
end
```

Then, add a `before_action` to any actions we want to apply MFA to, like pushing a gem.

```ruby
# app/controllers/api/v1/rubygems_controller.rb
before_action :verify_with_otp, only: %i[create destroy] 
```

That seems well. But we still need a way to know if current user has enabled multi-factor authentication, for clients. So a new controller for this.

```ruby
# app/controllers/api/v1/multifactor_auths_controller.rb
class Api::V1::MultifactorAuthsController < Api::BaseController
  before_action :authenticate_with_api_key
  before_action :verify_authenticated_user

  def show
    respond_to do |format|
      format.any(:all) { render plain: @api_user.mfa_level }
      format.json { render json: { mfa_level: @api_user.mfa_level } }
      format.yaml { render yaml: { mfa_level: @api_user.mfa_level } }
    end
  end
end
```

I did not intend to add a _plain_ type response. But when I went through code in RubyGems, I found a big part of them only uses response text without parsing them. So I just let it go. These may be handled along with [Issue #1683](https://github.com/rubygems/rubygems.org/issues/1683).

## Work on RubyGems

Take `PushCommand` as an example. Workflow here should be:

- Detect if current user has enabled MFA for write.
- If not, do not request or send OTP.
- Otherwise, see if OTP provided through command line options `--otp`. (e.g. `gem push hello-0.0.0.gem --otp 123456`)
- If OTP not provided, ask for input.
- Send OTP along with other header fields.

See the changes.

```ruby
# lib/rubygems/commands/push_command.rb
add_option('--otp CODE', 'Digit code for multifactor authentication') do |value, options|
  options[:otp] = value
end

# Above is part of initialize

def execute
  @host = options[:host]
  sign_in @host
  run_mfa_check
  send_gem get_one_gem_name
end

# Below is part of send_gem

response = rubygems_api_request(*args) do |request|
  request.body = Gem.read_binary name
  request.add_field "Content-Length", request.body.size
  request.add_field "Content-Type",   "application/octet-stream"
  request.add_field "Authorization",  api_key
  request.add_field "OTP", options[:otp] if need_otp?
end
```

`run_mfa_check` is in `GemcutterUtilities`.

```ruby
# lib/rubygems/gemcutter_utilities.rb

# Require user for extra OTP code if multifactor authentication is enabled.
def run_mfa_check
  return unless need_otp?
  unless options[:otp]
    say 'This command needs an extra OTP code for multifactor authentication.'
    options[:otp] = ask 'Code: '
  end
end

# Fetch user's multifactor authentication settings and return if an extra OTP code is needed.
def need_otp?
  unless @mfa_level
    response = rubygems_api_request(:get, 'api/v1/multifactor_auth') do |request|
      request.add_field 'Authorization', api_key
    end
    # For compatibility to Gemcutters without mfa support
    @mfa_level = case response
                 when Net::HTTPNotFound
                   'no_mfa'
                 else
                   with_response(response) { |resp| resp.body }
                 end
  end
  @mfa_level == 'mfa_login_and_write'
end
```

If a `NotFound` returns, we can conclude that the Gemcutter server does not support MFA. This works well for command line manually test. But a problem occurs when testing.

Testing RubyGems should not require a real Gemcutter instance. So it uses a _fake_ remote fetcher for mocking responses. But adding `@fetcher.data["#{Gem.host}/api/v1/multifactor_auth"] = ['no_mfa', 200, 'OK']` doesn't help to suppress errors. After this resolved, I will send a pull request to RubyGems.

P.S. Pushing gems consumes a lot of time unexpectedly because of checking newest RubyGems version. It takes so long that even OTP with drifts expires. I currently think is just my network problem.

## MFA Level Change

I turned the _Disable_ button into _Update_. Now there's a `select` element for selecting MFA levels. People maybe confused with that. So I plan to add a link to RubyGems document about multi-factor authentication after prompt text.

What it looks like:

![Planned MFA Level Change](https://ecnelises.com/images/change-mfa-level-plan-2.png)

## Plan

In the following week, I will continue to do work of client.

- Sign in in `gem` now requires no OTP even if MFA turned on.
- Changes to `OwnerCommand` are not finished.
- Tests are not finished.

Also, for the un-merged pull request about API and level change, other issues (if exists) on V2 API, and documents.