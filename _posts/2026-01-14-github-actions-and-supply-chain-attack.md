---
layout: post
title: "GitHub Actions and Supply Chain Attack"
date: 2026-01-14 00:00:00 +0000
---

Worst nightmare: you implement everything according to official docs, only to get hacked anyway. Meet the supply chain attack. But where to expect problems?

Let's take an example: I want to make sure that developers in my team don't push tokens and passwords to GitHub. And I don't want to pay for GitHub Advanced Security. So I decide to use [GitLeaks](https://github.com/gitleaks/gitleaks). It has 24.6k stars on GitHub, releases are frequent, and it is available as a pre-commit hook and [GitHub action](https://github.com/gitleaks/gitleaks-action). Nice! So I write action workflow:

{% raw %}
```yaml
name: gitleaks
on:
  pull_request:
jobs:
  scan:
    name: gitleaks
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
```
{% endraw %}

I make this check required to merge to main branch and secrets no more leak to GitHub. Also Dependabot creates pull requests to update action version when new release is available and raises alerts when CVEs were reported for action. Great!

But stop. Do I really trust the code that I run? I believe authors are good people and aware of security concerns, but things happen. What if their account is compromised and attacker pushes malicious code to the action repository? If malicious code runs in my workflow, attacker may not only steal source code but also get access to my infrastructure using stolen repository secrets. Stakes are high. So, I check what's going on there.

```v2``` in ```gitleaks/gitleaks-action@v2``` means "v2 tag". If attacker manages to push malicious code to the action repository and overwrite a tag, my workflow will run that code. I can use commit hash instead of version tag:

```yaml
- uses: gitleaks/gitleaks-action@<commit-hash>
```
Hashes are unique, so I can be sure that code won't change unexpectedly. Dependabot can update hashes, but unfortunately it doesn't raise alerts if action is pinned to commit hash (see [docs](https://docs.github.com/en/actions/reference/security/secure-use#monitoring-the-actions-in-your-workflows)). Tradeoffs everywhere. In October 2025 GitHub rolled out immutable releases, hope actions authors will start using them soon.

Time to go deeper. What action really does? It [downloads](https://github.com/gitleaks/gitleaks-action/blob/bf2dc8e55639c1e091e9b45970152e4313705814/src/gitleaks.js#L46) GitLeaks from GitHub releases:
```js
downloadPath = await tc.downloadTool(
    gitleaksReleaseURL,
    path.join(os.tmpdir(), `gitleaks.tmp`)
);
```
Good that by default release version is [pinned](https://github.com/gitleaks/gitleaks-action/blob/bf2dc8e55639c1e091e9b45970152e4313705814/src/index.js#L138):
```js
let gitleaksVersion = process.env.GITLEAKS_VERSION || "8.24.3";
if (gitleaksVersion === "latest") {
  gitleaksVersion = await gitleaks.Latest(octokit);
}
```
But GitLeaks releases on GitHub are not yet immutable, so files attached to releases can be overwritten. Still a room for supply chain attack. And GitLeaks action is not bad or unusual. For example, [reviewdog/action-template](https://github.com/reviewdog/action-template) is a template for new ReviewDog actions and it [downloads and executes](https://github.com/reviewdog/action-template/blob/9ac5749171403833526151728ad52375d1cd1b13/Dockerfile#L10) script from GitHub repository:
```dockerfile
RUN wget -O - -q https://raw.githubusercontent.com/reviewdog/reviewdog/fd59714416d6d9a1c0692d872e38e7f8448df4fc/install.sh| sh -s -- -b /usr/local/bin/ ${REVIEWDOG_VERSION}
```
The version of the installation script was [pinned](https://github.com/reviewdog/action-template/commit/cd9196207be273ee3bb634d4622c001c5a3c4b5b) only after a successful attack, when a release tag was updated to malicious code in one of the repositories (see [GitHub](https://github.com/reviewdog/reviewdog/security/advisories/GHSA-qmg3-hpqr-gqvc) and [NIST NVD](https://nvd.nist.gov/vuln/detail/cve-2025-30154)). Another example is [PyCQA/bandit-action](https://github.com/PyCQA/bandit-action), which always [installs](https://github.com/PyCQA/bandit-action/blob/cd2700ff8e8a10b277288e068d0c207c614c46ee/action.yml#L85) latest version from PyPi:
```yaml
- name: Install Bandit
  shell: bash
  run: pip install bandit[sarif,toml]
```
If attacker manages to publish malicious version to PyPi, my workflow will run it. These are just two links in the supply chain. Tools themself can (and will) interact with third party services and they have their own dependencies. But I will stop here.

Understand me right: without a marketplace of ready-to-use actions, GitHub Actions would not be so powerful tool. And it would be a pity not to use such great tools as GitLeaks, ReviewDog, Bandit, and many others. Managing the full supply chain of every tool used is an enormous amount of work. So what to do?

Here are my thoughts. First, manage your risks. Target your efforts where value is highest. Second, solution should be multi-layered. Developers' environment should be separate from production. Tokens should have minimal required permissions and rotated regularly. GitHub runners should be isolated and single-use. Infrastructure as code should be used. Backups should be in place. Users should be trained. Emergency plans should be prepared and drilled. Third, security scans and software composition analysis should be enabled, alerts monitored, found problems fixed. GitHub Dependency tree and Dependabot are great help here. Fourth, new tools, including GitHub actions and software components, should be carefully evaluated before use. Do they have known vulnerabilities? Do they use immutable releases? Are they well maintained? Are they widely used? Do they follow security best practices? Do they pull dependencies from secure registries? Implement your own actions for critical tasks if there's no trustworthy third-party solution.
