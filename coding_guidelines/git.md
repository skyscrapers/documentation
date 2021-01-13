# Git

## Commits

**IMPORTANT**: Please read this great blog post: <https://chris.beams.io/posts/git-commit/>.

* Try to keep commits small
* Guidelines for good commit messages:
  1. Limit the subject line to 50 characters
  2. Capitalize the subject line
  3. Do not end the subject line with a period
  4. Use the imperative mood in the subject line: _If applied, this commit will \<your subject line here\>_. Examples:
     * If applied, this commit will _Fix hacaddy eips_
     * If applied, this commit will _Update survey-csv worker to queue:work (#396)_
     * If applied, this commit will _Remove deprecated methods_
     * If applied, this commit will _Deploy dashboard and OIDC proxies skyscrapers/runtime#3_
* Optional: Add a reference (GH issue) to your message. This automatically links your commit back to an issue. This is preferred when commiting something to the `main` branch
* Finally, make sure to commit correct file permissions (especially for Windows users 😉): `644` for all files unless it's meant to be an executable (eg. `skysginin`, `755`)

## Pull requests

We highly encourage you to make contributions that you are comfortable with. In order to work together on the same codebase we have some ground rules:

* All code changes should be tested, documented and follow the style of that project.
* Code changes should be pushed to a branch and created into a pull request that is reviewed by someone at Skyscrapers.
* When a pull request gets approved it needs to be applied before merging to the main branch (*Note:* don't forget to rebase the main branch before applying the changes to avoid rolling back any changes that were done before the creation of the branch).
* After everything is applied the pull request can be merged into the main branch and the branch used in the pull request should be removed.

## Branching & merging

* We use short-lived feature branches. Once approved through PR, this gets merged back into the main branch
* Naming: no preference since it's short lived
* Merging: We use *Squash & Merge*. Make sure the merge commit is meaningful (usually PR title)

```console
* 8c529e9 2018-07-31 | Change the survey-csv (#397)
* 8a230d3 2018-07-27 | doing modifications to staging containers
* 9416919 2018-07-26 | Update debugging_commands.md
```

Versus

```console
*   36f9784 2018-07-18 | Merge pull request #387 from skyscrapers/rm_old
|\
| * 8ac5bd1 2018-07-18 | Destroy old environment (tf)
|/
* a133bea 2018-07-18 | re-add upstream templat
```

## Pushing

* If you push changes and these are rejected to upstream changes, do a `git pull --rebase` before pushing again, so we keep a nice history

```console
* 8c529e9 2018-07-31 | Change the survey-csv (#397)
* 8a230d3 2018-07-27 | doing modifications to staging containers
* 9416919 2018-07-26 | Update debugging_commands.md
* 2b9fe65 2018-07-26 | remove memory constraints for workers on artisan level
* 32b5f4a 2018-07-26 | changing staging to multiaz (#395)
* 1e58b12 2018-07-26 | pass reservation parameters to the module
```

Versus

```console
*   5ba86d0 2018-08-01 | Merge branch 'main' of https://github.com/skyscrapers/customer
|\
| * 2f2d110 2018-07-31 | Bump k8s-base
| * 98520c8 2018-07-31 | Bump k8s-base
* | 08af4dd 2018-08-01 | demo1 chart update
|/
*   6f1751e 2018-07-31 | Merge branch 'main' of https://github.com/skyscrapers/customer
```

* Evidently avoid force pushing on the main branch, since this breaks history for everybody. There are some exceptions to this rule, mainly when a rewrite is needed for removing sensitive information. Always be transparent towards the whole team when doing this.

## Nice commands to know

* `git commit --amend` to rewrite your last commit message (before pushing)
* `git rebase -i HEAD~2` to start an interactive rebase with the last 2 commits (you can squash commits, rewrite, pick, drop, ...)
* `git config --global pull.rebase true` if you want to set rebase as the default option when pulling
