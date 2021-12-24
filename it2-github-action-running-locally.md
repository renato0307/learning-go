# GitHub actions running locally

From [github.com/features/actions](https://github.com/features/actions):

> GitHub Actions makes it easy to automate all your software workflows, now 
with world-class CI/CD. Build, test, and deploy your code right from GitHub.
Make code reviews, branch management, and issue triaging work the way you want.

If you never used GitHub actions please check the quickstart and videos 
available at [docs.github.com/actions](https://docs.github.com/actions).

We are going to use [github.com/nektos/act](https://github.com/nektos/act) to
run GitHub actions locally.

My main reason is described in the `act` README file:

> __Fast Feedback__ - Rather than having to commit/push every time you want to 
test out the changes you are making to your `.github/workflows/` files you can
use `act` to run the actions locally.

So, follow up the installations instructions.

Run `act -h` to ensure the installation is working.