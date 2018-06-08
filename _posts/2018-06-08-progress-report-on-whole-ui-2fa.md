# RubyGems GSoC Progress report on Jun 8th

This is a weekly report regarding my GSoC 2018 project, adding two factor authentication to existing RubyGems command line tool and RubyGems.org site, a worldwide gem hosting site supported by community.

This week, 2fa feature about UI part is almost done, except:

1. Unit tests about current feature (2fa toggle, user login OTP check)
2. A feature that toggles the feature (feature flags)
3. Tweaks on UI design and workflow

The first and second should be finished within this week, or at least before first round of evaluation.[^1] So it can be merged into `master` for insides to test, gradually updating the third.

At rest of this post, I'll talk about implementation details, possible issues, further work and my plan.

## Please login to continue

As we discussed and learned from what other communities has done, user's 2fa settings should be at one of three levels:

- `no-auth`, everythings works as before, no extra auth, no otp
- `auth-only`, of course 'auth' here means authentication process when login
- `auth-n-write`, this also checks otp when doing things that should be named with `!`, most of them is done by REST API

I've not currently add a choice for the three levels. Instead, user can simply choose to turn it on or off. Since API part is not finished, adding the third is meaningless. But that would be quick.

Previous post shows what 2fa settings page looks like. But since I modified them, it would be better to put them here.

![rubygems-org-2fa-setting-demo2.png](https://ecnelises.com/images/rubygems-org-2fa-setting-demo2.png)

![rubygems-org-recovery-codes-demo2.png](https://ecnelises.com/images/rubygems-org-recovery-codes-demo2.png)

The upper one is a page giving a QR-code showing necessary information for your authenticator app to generate 6-digit code. It's rendered as SVG. If you have some problem with the QR-code, you can also manually type parameters on the right into the app. Issuer name, the one before your email, is the HTTP host you are using. Just type the OTP code to continue.

If everything goes well, 10 recovery codes will be shown here. They are important for those lost their phones, every one can be used once in replacement of OTP. So I'm not sure if I should add a checkbox before continue. Click continue and you will be redirected to profile edit page.

## Implementation

I moved 2fa related controller methods into a single controller, see [this commit](https://github.com/rubygems/rubygems.org/pull/1729/commits/33f187254d917788c92b6ec8b367346b544f8fa6). To prevent user from re-enabling, there are two filters.

```ruby
def require_no_auth
  return if current_user.no_auth?
  flash[:error] = t('two_factor_auths.authed_no_access')
  redirect_to edit_profile_path
end

def require_auth_set
  return unless current_user.no_auth?
  flash[:error] = t('two_factor_auths.no_auth_no_access')
  redirect_to edit_profile_path
end
```

A user must have enabled 2fa before disabling it, and vice versa.

Another thing to implement is extra authentication when logging in. Logic here seems a little bit complex. So to make controller clean, I drop something into individual methods. Now our login action looks like:

```ruby
def create
  if verifying_otp? && !@user.otp_verified?(session_params[:otp])
    session[:mfa_user] = nil
    login_failure(t('two_factor_auths.incorrect_otp'))
  elsif !verifying_otp? && @user&.mfa_enabled?
    session[:mfa_user] = @user.id
    render 'sessions/otp_prompt'
  else
    do_login
  end
end
```

Both OTP checking and normal login request are sent to the same method. Originally, I thought I can use a `before_action` to wrap an action which requires OTP check. But since it needs one more submit and page redirect, combining the logic with action seems a better approach. [This commit](https://github.com/rubygems/rubygems.org/pull/1729/commits/eba6f1ba083fd2ac5b81d35a3010a491112e9cff) is about it.

## Issues & Further work

I just keep them here for memorization and possible discussion.

### Security

- Is it safe to keep 2fa keys/seeds without encryption in database?
- Is the 16-char key strong enough?
- To pass information between pages, I use session to store some critical data and clean them after the whole workflow is done. Is there any problem if user interrupt their operation, those data in session left?

### Further work

Some of things below may consume a lot of time. I'll work on them if I have extra time after 2fa done.

- Change way of RubyGems API signing, or use something like Json Web Tokens.
- Don't let API calls do sign in, see [Issue #1724](https://github.com/rubygems/rubygems.org/issues/1724)
- Support for U2F[^2]

## Plan

Google's newest timeline shows Final Evaluation will be Aug 14. So all coding and testing work should be done before August for time with documents. 2fa stuff on RubyGems.org API is expected to be simpler than what's on UI part since client program can interactively require otp code if received a 401 response. It is expected to be finished within June. After that I will hacking RubyGems for 2fa, and I'm not sure if there's something related to Bundler here.

If you have any advice, reviews or questions, please commit at [Issue #1725](https://github.com/rubygems/rubygems.org/issues/1725), [PR #1729](https://github.com/rubygems/rubygems.org/pull/1729), or share your comments in #rubygsoc-2fa channel on [Bundler Slack](https://slack.bundler.io/). Thanks.

[^1]: Chinese usually consider Monday as start of a week, I'm not clear about people from other countries, LOL
[^2]: U2F is a USB device authentication protocol, Firefox and Chrome supports it well now