# RubyGems GSoC Progress report on Jun 3rd

This is a weekly report on a project adding two factor authentication to RubyGems tool and RubyGems.org, a site for hosting RubyGems. This week my work focus on controllers and pages for the UI part. In my last post, I mocked up the UI of four occasions concerned with this feature. Now it's time to implement them.

## Routes

Since we have discussed occasions in the mock-ups, it's natural to have five actions:

- Showing the qr-code and recovery codes.
- Enabling two factor authentication.
- Disabling two factor authentication.
- OTP requirement.
- OTP confirmation.

Just in `config/routes.rb` UI part:

```ruby
resource :two_factor_auth, only: %i[new create destroy]
```

So we can use `/two_factor_auth/new` to access the 2fa settings page.

## Controllers

Every time we generate different random 2fa seeds and recovery codes when user try to set up 2fa. We store them in session, so we can have it when user confirms to enable 2fa while user cannot change these secrets by themselves.

```ruby
seed, @recovery = User.generate_mfa
session[:mfa_seed] = seed
session[:mfa_recovery] = @recovery
```

And when the site receives `POST` request from user to enable 2fa, the sessions will be cleared.

```ruby
seed = session[:mfa_seed]
recovery = session[:mfa_recovery]
session[:mfa_seed] = session[:mfa_recovery] = nil
totp = ROTP::TOTP.new(seed, issuer: 'RubyGems.org')
```

For the disabling process, user just inputs current OTP to send `DELETE` request to disable it.

```ruby
def destroy
  if current_user.otp_verified?(params[:otp])
    flash[:success] = t('.disable_success')
    current_user.disable_mfa!
  else
    flash[:error] = t('profiles.mfa_enable.otp_auth_failed')
  end
  redirect_to edit_profile_url
end
```

We should ensure that when a user tries to enable 2fa, he/she should have not enabled it yet. And vice versa.

```ruby
before_action :require_no_auth, only: %i[new create]
before_action :require_auth_set, only: :destroy
```

These means we can turn on/off 2fa settings now. See [my commit](https://github.com/ecnelises/rubygems.org/commit/1ab422a960d855ffe3673fcb420f4a373759b98e).

## More issues

- User cannot set auth level now.
- No _real_ auth phase in login and other stuff.
- When signing the issuer, I hard-coded 'RubyGems.org' into it. It should be replaced by the actual deployment host name.

These should be further discussed with my mentor. Next Sunday (Jun 10th) is around GSoC 2018's mid-term evalution time. I expect to finish site part (2fa of UI & API) before that deadline. Since my code base currently does not have any tests on it, it cannot be merged into main code base. But I still plan to open a pull request (labelled with _WIP:_) to show the progress.



