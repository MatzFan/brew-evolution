# Warn on invalid brew install options
## Introduction
Warn the user if invalid formula option arg(s) are passed to `brew install <formula>`

## Motivation
Currently `brew install foo --with-invalid_option` does not warn the user that `invalid_option` was ignored. Typos, or a user misunderstanding the valid options for \<formula\> are two examples of how this can occur. Install then proceeds by silently ignoring `invalid_option`, so the user may assume that whatever install option(s) he/she intended have been included in the build.

Originally raised in 2011 as a feature request in [legacy-homebrew #8937](https://github.com/Homebrew/legacy-homebrew/issues/8937).

## Proposed solution
A post-expansion hook should perform a validation check on each build option parsed from the command line. Interactive confirmation that the user wishes to proceed should be required if invalid option(s) are detected. E.g:

`$ brew install foo --with-bat` should result in a warning level alert, along the following lines and await user confirm before proceeding:

```
Warning: '--with-bat' is not a valid option for formula 'foo' - did you mean '--with-bar'?
Do you wish to proceed and install foo ignoring invalid option(s) Y/n?
```

Such functionality will improve the user experience and provide a safety net for users to ensure that incorrect/unintended build options are intercepted.

## Detailed design
Invalid build options for a given formula can be trivially derived in `BuildOptions`:
```
def invalid_options
  used_options ^ @args
end
```

Implementation should be limited to formula(e) passed as command line args after `brew install`. It should not attempt to validate the build options for any immediate or recursive dependencies - see @jacknagel's [comment](https://github.com/Homebrew/legacy-homebrew/issues/8937#issuecomment-9033839).

It must be assumed that a user passing build options to the install command for multiple formulae intends for them all to apply to each one. Ideally then, any incompatibilities would be picked up in a single warning to the user *before* the formula install loop - i.e. at [`perform_preinstall_checks`](https://github.com/Homebrew/brew/blob/c45119de75e70f32e3b3fdcccb210a88282a2f26/Library/Homebrew/cmd/install.rb#L156). However, as each formula's build options are currently parsed within the install loop a more practical solution would be to issue a warning for each formula. Also, options conditional on other formula being installed can only be properly validated immediately before install - see end of this [comment](https://github.com/Homebrew/legacy-homebrew/issues/8937#issuecomment-9024193).Thus the following scenario would be possible:

```
$ brew install foo bar --with-valid_foo_option
==> Downloading...

...foo install succeeds

==> Summary
🍺  /usr/local/Cellar/foo/0.0.1

Warning: '--with-valid_foo_option' is not a valid option for formula 'bar' # no typo suggestions this time
Do you wish to proceed and install bar ignoring invalid option(s) Y/n?

$ Y
...bar install proceeds

```

Typo suggestions, as illustrated with `--with-bat`, are a 'should have' feature & could be implemented using an algorithm such as Levenshtein distance (as used in [RubyGems](https://github.com/rubygems/rubygems/blob/58e2de2ea1f1a78e7548521206863ed3ba0d3e8f/lib/rubygems/text.rb#L40)). Homebrew may already use such an algorithm [TBC].

The option to suppress such warnings may be desirable for users, particularly in non-interactive environments. A `--no-options-validation` flag could be added to the `brew install` command to achieve this.

## Alternatives considered
The proposal is to make this functionality default behaviour. An alternative may be to provide it via a `--validate-options` flag or similar. However, as it's intended purpose is to pick up errors in user instructions it seems reasonable to make it the default.

It would be possible (easier) to error & exit instead of warn with user interaction. As user interaction in Homebrew seems to only be employed in dev commands (`brew create`) & debugging, it may be that the community decides that this is the preferable approach.
