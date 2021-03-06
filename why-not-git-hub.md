# Why Not GitHub?






[](https://i.imgur.com/lWvuVPd.jpg)



A common suggestion that is brought up from time is to host GHC development at [
GitHub](https://github.com/) instead of maintaining our own infrastructure for Git repository hosting, code-review, developer documentation/wiki, and issue tracking. This page serves as reminder for what's holding back a migration to GitHub currently.


## Benefits of migrating to GitHub


- Reduce maintenance cost in light of our already limited resources.

- Potential contributors most likely have a GitHub account already and are used to contributing on GitHub, reducing disincentives to contributing.

  - Projects like Cabal saw an uptick in contributions after moving to GitHub.

- GitHub's pull-request-based work-flow is well-known in the Haskell community (and the wider developer community), not the least due to the majority of Haskell packages being developed on GitHub already

- GitHub is well-integrated with services like [
  Travis-CI](https://travis-ci.org) (which allow to automatically validate pull-requests)

- Trac poses a higher threshold for new contributors due to its interface being more complex and requiring registering yet another account.

- Trac uses its own wiki markup syntax (as opposed to Markdown everyone else is using).

- See also: [WhyNotPhabricator](why-not-phabricator).

## Drawbacks of migrating to GitHub


### Code review


- ~~Every inline comment results in a separate email~~ (GitHub added support for multi-part code reviews)

- On Github, [\#1234](http://gitlabghc.nibbler/ghc/ghc/issues/1234) doesn't link to Trac ticket 1234. You'd have to say: [
  https://ghc.haskell.org/ticket/1234](https://ghc.haskell.org/ticket/1234).

- No possibility to do Post-Commit Code Reviews (i.e. Audit in Phabricator).

- No possibility to do Rule-based Code Reviews (i.e. Herald in Phabricator)

- No review-queue (i.e. list of pull-requests that haven't been reviewed yet) 

- Doesn't show code reviews (pull-requests) that touch the same file.

- 'git push' doesn't run the linter locally, while 'arc diff' does.

- No possibility to mark inline comments 'Done'.

- No way to review diffs with more than 1500 lines.

### Git repository


- GitHub lacks several things we already use. For example, there is **no way to add pre commit hooks** to repositories that ban commits containing whitespace, trailing spaces, and other `lint` errors. [
  https://git.haskell.org](https://git.haskell.org) automatically enforces this to help keep new code tab-free. GitHub has no alternative to this.

- We also use this facility to **keep Git submodules sane**: as of today, [
  https://git.haskell.org](https://git.haskell.org) will not let you commit a *dangling submodule reference* to [
  ghc.git](https://git.haskell.org/ghc.git). You must push the corresponding submodule code first, so the top-level repository never breaks. This is also not possible with GitHub and has been a historical error source for developers.

### Issue Tracking


- **Migration cost.** GHC is probably one of the largest Trac installations around at nearly 10.000 tickets, a gigantic wiki, and tons of meta-data and a **lot** of users. Preserving the necessary meta-data, rewriting intra-wiki links, references, and preserving everything is just going to be a ton of work. GitHub doesn't even have a proper "import" facility.

- Morever, **a GitHub migration would be lossy**, as the data would have to be imported via the current GitHub API which for one doesn't allow to set all meta-data, like even timestamps, ticket/comment authors. And some of our ticket meta-data fields do not even have any corresponding concept in GitHub's issue tracking data-model.

- Trac's wiki syntax (which is available in ticket comments as well) is extensible and allows for dynamic content generation via [WikiMacros](wiki-macros), such as dynamically generated tables of tickets.

### Continuous Integration


- The free for open-source version of Travis-CI only provides limited resources. Currently this means we can only run `./validate --fast` (this could be a feature, to press us into not letting the build and the testsuite become slower).

- If a pull request contains multiple commits, only the last commit gets validated. This can (and will) result in some commits not passing validation, making `git bisect` more difficult to use.

- Any kind of integration with a **CI system**, at some level, is going to require custom infrastructure on our side, so we can't rely on Travis-CI alone. (thomie: I don't understand this one)

### Other issues


- We would still need to maintain our own server so that Git pushes can interface with Trac, and the **mailing notifier**.

- Github is not open-source, so we can't fix any (future) issue we might have with it.

- Some people will have a different handle on GitHub then they do on Trac.

## Related mailing-list discussion threads


- [
  "Any interest in moving GHC issues / wiki to github?" (Jul 2013 @ ghc-devs)](http://thread.gmane.org/gmane.comp.lang.haskell.ghc.devel/1444)
- [
  "Phabricator for patches and code review" (Jun 2014 @ ghc-devs)](http://thread.gmane.org/gmane.comp.lang.haskell.ghc.devel/4829/focus=4861)
- [
  "I think the amout of contributions would increase significantly if GHC migrated to GitHub and started accepting pull requests." (Sep 2014 @ reddit)](https://www.reddit.com/r/haskell/comments/2hes8m/the_ghc_source_code_contains_1088_todos_please/ckrzyec)
- [
  GitHub pull requests](https://mail.haskell.org/pipermail/ghc-devs/2014-October/006523.html)
- [
  Proposal: accept pull requests on GitHub](https://mail.haskell.org/pipermail/ghc-devs/2015-September/009773.html)
- [
  GHC Development: OutsideIn](https://www.reddit.com/r/haskell/comments/4isua9/ghc_development_outsidein/d32979t)
