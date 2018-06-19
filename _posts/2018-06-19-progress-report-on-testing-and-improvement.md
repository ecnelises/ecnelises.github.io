# RubyGems GSoC Progress report on Jun 19th

This is a weekly report regarding my GSoC 2018 project, adding two factor authentication to existing RubyGems command line tool and RubyGems.org site, a worldwide gem hosting site supported by community.

Last week, I listed three points left to be done, including unit tests, feature flags and some tweak, along with several commits for advice from mentors on the [Pull Request](https://github.com/rubygems/rubygems.org/pull/1729).

## Unit tests

I added tests (most focusing on controllers regarding two-factor authentication) for my implementation on extra authentication. For details, see [my commit](https://github.com/rubygems/rubygems.org/pull/1729/commits/77977dd6e071bbbb10df960a78c0ea45141ae0be). I should admit that I do not have much experience or knowledge on Rails app testing. And last internship left me a awful impression on Rsepc. But we don't use it here. And I may find it not so bad after I tried writing these test cases :)

Two things should be pointed out that every setup path is executed independently for ever y test case in `should` clause, and we must reload some model objects if we want to see how it's affected by controller actions.

Another detail that seems a little bit interesting is generating 'false' otp codes. Here I write:

```ruby
context 'when otp code is incorrect' do
  setup do
    wrong_otp = (ROTP::TOTP.new(@user.mfa_seed).now.to_i.succ % 1_000_000).to_s
    delete :destroy, params: { otp: wrong_otp }
  end

  should respond_with :redirect
  should redirect_to('the profile edit page') { edit_profile_path }
  should 'keep 2fa enabled' do
    assert @user.reload.mfa_enabled?
  end
end
```

Parse it to integer and increment it, back to string. Even considering drifts for both 'previous' and 'next' otp code, the possibility that the code is valid is negligible, since TOTP uses a special hash algorithm.

Also, [@indirect](https://github.com/indirect) pointed out that functional tests are usually for controllers and unit tests are for models, in context of Rails testing. Integration tests are in a higher level, from perspective of workflow. Now I'm rejoiced that I'm not handling tests for a front-end project with heavy JavaScript...

So here, integration tests and tests for 'mfa should be disabled by default' are not tested yet.

## Feature flag

We want to merge this feature into main stream quickly as soon as it meets requirement for code quality and security. More issues can be discussed more efficiently if some people can experience it. So we need a way to roll out the feature to people and hide it by default.

For complex apps, we may use some dedicated gems like `rollout` for this stuff. It can use some rules defined by developer (also has conventions over configuration, of course) like select only 5% of users. But here this approach may introduce too much extra complexity.

So we hide switch of the feature into cookies. Just set `mfa_feature` to `true` to turn it on:

```ruby
def mfa_enabled?
  cookies.permanent[:mfa_feature] == 'true'
end
```

That's cool! If it's not set to `true`, nothings happens even if you turn mfa on in database! If you are using JavaScript console on your browser for editing cookies, remember to set a path or domain for it, otherwise it may not work.

## Drifts

I also implemented drifts and `last_otp_at` feature into current branch. Sometimes a user may be frustrated for a just expired otp code. Or due to some network issue, user's phone time are not totally symmetric with that on server. It's convenient for user to input 'previous' and 'next' otp code for each prompt. `rotp` provides us with this.

There's another security issue that a user may use an otp code for several times. To prevent this, it's necessary to record last successful otp check time. See [rotp manual](https://github.com/mdp/rotp#preventing-reuse-of-time-based-otps) for detailed description about this.

I add a new attribute `last_otp_at` to `users`. And now `User#otp_verified?` looks like this:

```ruby
def otp_verified?(otp)
  if no_mfa?
    true
  elsif mfa_recovery_codes.include?(otp)
    mfa_recovery_codes.delete(otp)
    save!(validate: false)
    true
  else
    totp = ROTP::TOTP.new(mfa_seed)
    last_verification = totp.verify_with_drift_and_prior(otp, 30, last_otp_at)
    if last_verification
      self.last_otp_at = Time.at(last_verification).utc.to_datetime
      save!(validate: false)
      true
    else
      false
    end
  end
end
```

Note the `utc` from time.

## Tweaks

- Store user's name in session, instead of id. [See commit](https://github.com/rubygems/rubygems.org/pull/1729/commits/4660f8b52a0bec44547ae9a0e9d7c22d097d7d84)
- Move all i18n entry and content text from '2fa' or 'tfa' to 'mfa'.
  - `TwoFactorAuths` are now `MultifactorAuths`
  - `mfa_level` enum cases are now `no_mfa`, `mfa_login_only` and `mfa_login_and_write`.
- Move login OTP check into a single method.
- Some method names are changed to more readable.
