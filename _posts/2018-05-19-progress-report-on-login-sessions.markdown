# Progress report on May 19th
This is my technical journal on my progress of adding two factor authentication to [RubyGems](https://github.com/rubygems/rubygems) tool and its respective [hosting site](https://rubygems.org), along with other stuff regarding authentication or authorization of them. This is also a [Google Summer of Code 2018](https://summerofcode.withgoogle.com) project.
## Login sessions
This week I encountered issue [#1724](https￼://github.com/rubygems/rubygems.org/issues/1724), saying any push operation will make other login states reset. The problem is easy to re-produce. Pushing gem is a _POST_ request to `/api/v1/rubygems`, a `create` action, which just creates a new `Pusher` object, validate it, and save it if no problem raised. Reason should lies in authentication process defined in `ApplicationController` triggered using `before_action`. When I check logs when every push request received, I found it.
User authorization work in RubyGems.org is implemented through [Clearance](https://github.com/thoughtbot/clearance), a light-weight library compared with full-featured ones like Devise. User login state is kept by an attribute stored in users table named `remember_token`. Every time when a new login attempt is successful, a new token will be generated and replace previous token in database.
Of course, this is not the best practice. In an ideal user authentication system, you can check all your active logins and you can disable any of them, just like what you see in your FaceBook account... But such things can only make a quite simple site far more complicated. It provides API keys for RESTful API requests authorization.
### How gem do logins
We’re quite familiar with how login process happens in browser. Now we should take a look at how login executed when doing `gem push` command. We can see that every `gem` command has a respective class in path `lib/rubygems/commands/` directory. Let’s take a look at `PushCommand`. Using newest master branch will complain a version non-match error, I use tag `v2.7.7` here, since code here are not changed often.
```ruby
# lib/rubygems/commands/push_command.rb
def execute
  @host = options[:host]
  sign_in @host
  send_gem get_one_gem_name
end

def send_gem name
  # ... version checks
  gem_data = Gem::Package.new(name)
  # ... command line prompts
  response = rubygems_api_request(*args) do |request|
    request.body = Gem.read_binary name
    request.add_field "Content-Length", request.body.size
    request.add_field "Content-Type",   "application/octet-stream"
    request.add_field "Authorization",  api_key
  end
  with_response response
end
```
We want to let it only use API keys, but here it are surely using API keys for request, not using basic authentication by username and password. Now we check function `sign_in` for detail.
```ruby
# lib/rubygems/gemcutter_utilities.rb
def sign_in sign_in_host = nil
  sign_in_host ||= self.host
  return if api_key
  pretty_host = if Gem::DEFAULT_HOST == sign_in_host then
                  'RubyGems.org'
                else
                  sign_in_host
                end
  say "Enter your #{pretty_host} credentials."
  say "Don't have an account yet? " +
      "Create one at #{sign_in_host}/sign_up"
  email    =              ask "   Email: "
  password = ask_for_password "Password: "
  say "\n"
  response = rubygems_api_request(:get, "api/v1/api_key",
                                  sign_in_host) do |request|
    request.basic_auth email, password
  end
  with_response response do |resp|
    say "Signed in."
    set_api_key host, resp.body
  end
end
```
### How session refreshed
Now we’ve figured out that `gem` use `sign_in`, sending username and password to server to get API key. If the key already is there, just return it. Turn eyes into what the server does.
```ruby
# config/initializers/clearance.rb
class Clearance::Session
  def current_user
    return nil if remember_token.blank?
    return @current_user if @current_user
    user = user_from_remember_token(remember_token)
    @current_user = user if user&.remember_me?
  end

  def sign_in(user)
    @current_user = user
    cookies[remember_token_cookie] = user && user.remember_me!
    status = run_sign_in_stack
    unless status.success?
      @current_user = nil
      cookies[remember_token_cookie] = nil
    end
    yield(status) if block_given?
  end
end
```
The action `remember_me!` makes app updates user’s remember token. For testing purpose, comment it out and things go as we want - I will not lose my browser session after a command line push. But it cannot be really used because browser logins will also use this method. One solution I think of is override `current_user` method in `Api::BaseController` and not use `sign_in` in API response actions.
## OTP and QR-code
My main work on GSoC is adding two factor authentications. From perspective of databases, what we need to add includes:
* Seed for OTPs
* Authentication level (as we mentioned, auth-only & auth-and-write)
* Recovery (or backup) codes
  From perspective of users:
* QR-code for me to scan in authenticator app
* Recovery codes
* OTP when required
  So two things are important: QR-code and OTP generator. We have two good gems for them: [ROTP](https://github.com/mdp/rotp) and [rQRCode](https://github.com/whomwah/rqrcode). GitLab uses devise along with a plugin to enable 2FA feature. I once read its code, found that it also relies on ROTP. Here I take a test to show how we can draw QR-code scannable for authenticator apps.
```ruby
# app/controllers/profiles_controller.rb
def two_factor_authentication
  @seed = ROTP::Base32.random_base32
  @text = ROTP::TOTP.new(@seed, issuer: 'RubyGems.org').provisioning_uri(current_user.email)
  render inline: RQRCode::QRCode.new(@text, level: :l).as_svg
end
```
Now we see a QR-code in page, which can be scanned.
![qrcode.png](https://ecnelises.com/images/rubygems-test-qrcode-show.png)
## What’s next
There’s three weeks before the first evaluation. More real work should be done this week.
* Open another RFC in GitHub on workflow
  * A page requiring OTP or recovery code after click ‘Sign in’ if 2FA enabled.
  * A link to 2FA settings in profile page. (Disabled it also requires OTP)
  * Pushing and changing owner of gems requires OTP on command line if enabled 2FA.
* Make model migrations. Mainly on user. Three new attributes.
  * OTP seed
  * 2FA level (an enum)
  * Recovery codes (since we’re using Postgres, this is an array in schema)
* Create 2FA settings page.
  * Layout based on profiles page.
  * QR-code and recovery code are put together.
  * A user must input current OTP to enable it.
* Look for a better solution to solve `gem` login problem. Since OTP authorization should be around these methods I mentioned above, working on this issue really helps.