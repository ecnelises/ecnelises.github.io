# RubyGems GSoC Report for UI Part

This is an article about progress of adding multi-factor authentication (MFA) into [RubyGems.org](https://rubygems.org) site. There have been several reports for different parts of implementation, you can check them out [here](https://ecnelises.github.io). Currently, at early July 2018, work for the site except API (we often call them UI part) has already been finished and merged into RubyGems' main develop branch, see [PR#1729](https://github.com/rubygems/rubygems.org/pull/1729).

## What is MFA and why we need it?

This topic deserves another article. Authentication of users' identities has been always a problem since Web 2.0 or earlier internet services came into being. Simplest way of logging in is using the combination of username (or email) and password. Your password is static and if someone stole it your account would be got. Considering every time you'll type the same password or sometimes you have to rent your account to your friend (like Steam account), password authentication is quite vulnerable.

There are many solutions now. An impressive one to me is [TypingDNA](https://www.typingdna.com/), which authenticates you by how to type. Security controls used by banks and some network game providers in the early times are also a good but inconvenient solution. The MFA we are talking about now is an economic choice. As I wrote in past posts, MFA is an authentication system as addition to traditional auth methods by using one-time-passwords (OTP), which solves the two problems I raised above. In real life we usually have three types of MFAs:

### By an algorithm for symmetric OTPs

The first one is almost synonymous to TOTP and HOTP auth system. They are discribed in detail in their respective RFC documents, maybe I'll use a single article for introduction to how they work.

### By using SMS in mobile phones

The second one is used by Telegram, Signal, WhatsApp and many other mobile-oriented apps. Note that messages sent to your phone is quite possible to be modified without you informed. So some apps requires you to send message to them to do this authentication due to difference in difficulties between modifying incoming messages and sending-out messages. SMS is still costy, and inconvenient for desktop users, nevertheless.

### By hardware

This is quite similar as the _security controls_ I said in above text. Currently there's a universal standard named U2F and products like Yubikey or other cheaper ones. Chrome, Opera and Firefox now supports it well. I'll discuss it in further articles.

Here we choose time-based one-time password (TOTP) as the solution, for it's simple, reliable and used by many large organizations and projects. Many people use mobile auth apps like _Authy_ or _Google Authenticator_. Adding MFA for them just means scanning another QR-code.

## Development detail

The development starts from a research on gems for the functionality. If you search _two factor authentication_ in GitHub, a list of multi factor auth related gems will be shown, but some of them are no longer maintained and most of them are extension of _devise_. After checking source code of them, I found they all depends on _rotp_ for the main totp feature.

So I added _rotp_ into the project first, along with a gem showing QR-code named _rqrcode_. You can refer to [this post](https://ecnelises.github.io/2018/05/progress-report-on-login-sessions) for more details. 

### Why not a separate gem?

Actually I was planning to write a separate gem for the multi-factor auth functionality aiming at integrating with Clearance. But I didn’t do that for several reasons.

- The gems I found first are most extensions for Devise. Devise is a great and famous gem for its full feature and maturity. It takes your controller, model or even some part of views over for the login and registration process. So extending the logic is easier, than doing such things onto a gem that only focuses on much fewer stuff.
- What if I wrote a wrapper (such as `before_action`) for the extra authentication work? Seems it’s not a good choice, since each site may have its own design and workflow, and some sites may choose a no-refreshing approach using JavaScript, while developer of other sites may carry a belief that the site should run well even if JavaScript is turned off. In my implementation, the OTP prompt is another render of page. A filter in Rails cannot handle such logic related to multiple redirects well.
- The key is to give a well-implemented multi-factor authentication feature to RubyGems.org. So making everything works is the first level of concern, especially for our site is not complex, comparing with big ones like GitLab or Discourse.

But this is still a good idea. And I may work on it after these done.

### Structure of the addition

The final pull request merged contains 18 commits, having about 800 lines of code. Main changes includes:

- New gems added into Gemfile and Gemfile.lock.
- A new controller about multi-factor authentication stuff added, named `MultifactorAuthsController`. It has `new`, `create` and `destroy`, three actions. See [this post](https://ecnelises.github.io/2018/06/progress-report-on-2fa-settings) for detail.
	- When user accesses `multifactor_auths#new` from profile settings page, a QR-code will be shown (of course, it does a pre-check for current settings on the auth) and the key is stored in session. User can post a form with a digit generated from the key.
	- If the digit code (OTP) is correct, the auth is enabled and recovery keys will be shown to user. Otherwise, it fails and key in session will be cleared.
	- `multifactor_auths#destroy` handles users’ request on disabling the auth.
- In `sessions#create`, we checks auth settings of current user. If the extra auth is enabled, OTP will be required. That will be posted to a single method named `mfa_create`.
- Methods about authentication and auth setting state enums are added into users model. See [this work report](https://ecnelises.github.io/2018/05/progress-report-on-pages).
- A new database migration adding key, state, recovery codes of multi-factor auth, along with `last_otp_at`, an attribute for preventing a OTP authenticated many times.
- Certainly, the related pages are also added.
- Unit tests for the controller and model, with integration tests, are written. See [report on Jun 19th](https://ecnelises.github.io/2018/06/progress-report-on-testing-and-improvement).

### What’s next

[PR #1753](https://github.com/rubygems/rubygems.org/pull/1753) is next thing to be finished. And something about [push command](https://github.com/rubygems/rubygems/blob/master/lib/rubygems/commands/push_command.rb) and [owner command](https://github.com/rubygems/rubygems/blob/master/lib/rubygems/commands/owner_command.rb). I’m not sure if Bundler will be affected so I will keep contact with my mentor.

## Gains

- Actually, I did not write serious tests for my projects before. This helps me understand how to consider test branches, what to assert (or refute) and, the importance of testing. It really reduces anxiety when adding new things.
- Open-source projects are somewhat in a loose communication. The first thing is to figure things out by self. And it’s good to remember ‘you can you up’.
- The community has fewer active people than I thought! So every contribution may be very important.
- It’s necessary to let others (especially mentors) know your progress.

## Acknowledgement

Thanks for my mentors, [@sonalkr32](https://github.com/sonalkr132) and [@indirect](https://github.com/indirect). They answered my silly questions and reviewed my code. I also want to say thanks to [GitLab](https://github.com/gitlabhq/gitlabhq), for they are a good model showing what is a good multifactor-auth workflow. (In fact, I heard of the noun from GitLab) Some of other participants of Ruby GSoC this year, I talked with them, knew how different people interacts with their mentors. This also helped me.