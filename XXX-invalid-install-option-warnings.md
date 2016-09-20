# Warn on invalid brew install options
## Introduction
Warn the user if invalid formula option arg(s) are passed to `brew install <formula>`

## Motivation
Currently `brew install foo --with-invalid_option` does not warn the user that `invalid_option` was ignored. Typos, or a user misunderstanding the valid options for \<formula\> are two examples of how this can occur. Install then proceeds by silently ignoring `invalid_option`, so the user may assume that whatever install option(s) he/she intended have been incuded in the build.

Originally raised in 2011 as a feature request in [legacy-omebrew #8937](https://github.com/Homebrew/legacy-homebrew/issues/8937).

## Proposed solution
A post-expansion hook should perform a validation check on each option parsed from the command line. Interactive confirmation that the user wishes to proceed should be required if invalid option(s) are detetected. E.g:

`$ brew install foo --with-bat` should result in a warning level alert:

```
Warning: '--with-bat' is not a valid option for formula 'foo' - did you mean '--with-bar'?
Do you wish to proceed and install foo ignoring invalid option(s) Y/n?
```

Such functionality will improve the user experience and provide a safety net for users to ensure that incorrect/unintended build options are intercepted.

## Detailed design
Implementation should be limited to single formula installs - i.e. one formula is specified after `brew install`. [Question: how are options for multiple formulae installs treated?]. It should also be limited to validating the options for that formula only, not it's immediate or recursive dependencies - see @jacknagel's [comment](https://github.com/Homebrew/legacy-homebrew/issues/8937#issuecomment-9033839).

Typo suggestions are a 'should have' feature & could be implemented using an algorithm such as [Damerau-Levenshtein](https://github.com/GlobalNamesArchitecture/damerau-levenshtein). Homebrew may already use such an algorithm elsewhere [TBC].

The option to suppress such warnings may be desirable for users, particularly in non-interactive environments. A `--no-options-validation` flag could be added to the `brew install` command to acheive this.

The code should be implemetned as a module so that in future it may be shared by other commands - for example `brew gist-logs`.

## Alternatives considered
The proposal is to make this functionality default behaviour. An alternative may be to provide it via a `--validate-options` flag or similar. However, as it's intended purpose is to pick up errors in user instructions it seems reasonable to make it the default.
