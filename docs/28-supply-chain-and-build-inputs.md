# 28 — Supply chain and build inputs

Every build pulls code written by people the project has never met: packages, pipeline
actions, base images and downloaded binaries. Each one runs with the rights of the thing
that pulled it — the developer's machine, the agent's box, the pipeline runner that holds
a deploy role.

An agent makes this worse in one specific way. **A model can name a package that does not
exist**, or a name that is one letter from a real one. Someone registers the invented name
and waits. The install then runs the attacker's code on the box, in the pipeline, and in
every clone. The common name for it is **slopsquatting**.

This file is the set of controls that close each path. They are independent: a project can
adopt them one at a time, and each one is worth having on its own.

Reference implementation: Reference A, ADRs "npm install scripts are off in every project;
an allowlist names the ones that run", "Every pipeline action is pinned to a commit SHA
and the updater updates the pins", "The pipeline roles trust a branch, and a plan uses a
read-only role".

---

## 1. No new direct dependency without a named decision

**An agent adds no new direct dependency unless the work item names the package, or the
owner approves it.**

This rule is written only. Reference A decided against a pipeline check for it (2026-10-23):
the agent and the reviewer are the check. A written rule with a review step beats no rule;
do not wait for a tool.

### 1.1 What counts as a manifest

A new direct dependency is a **new entry** in any file that names third-party code for a
build or a run:

- a new key in `dependencies`, `devDependencies`, `optionalDependencies` or
  `peerDependencies` of any package manifest;
- a new line in any requirements file, or a new entry in the project definition file;
- a new library line in a mobile dependency file;
- a new action in a pipeline `uses:` line;
- a new provider in an infrastructure `required_providers` block;
- a new entry in a build-tool `dependencies` or `plugins` block;
- a new dependency line in a native package specification;
- a new base image in a container `FROM` line.

**Any other file that names third-party code is a manifest too.** The rule covers a new
manifest kind at once, before the list names it.

A **version bump** of an existing entry is not a new dependency. It is still subject to
§ 3 and § 4. A change to an indirect dependency in a lock file only is outside this rule;
§ 3 and § 4 cover it.

### 1.2 What approval is

One of exactly two things:

1. A comment by the owner on the pull request that **names the package**.
2. The owner's answer quoted word for word in the pull request body, with its date.

An approval for one package does not approve a second package.

### 1.3 What the agent checks before it proposes a package

On the registry page itself — not from memory:

- [ ] the exact name, letter by letter, against the name it is about to write;
- [ ] the repository link, and that the repository is the source of the package;
- [ ] the weekly downloads;
- [ ] the first publish date;
- [ ] the maintainers;
- [ ] whether the package has install scripts.

**Prefer code that is already there.** Code already in the repository, or a package
already in the lock file, adds no new maintainer that the project must trust. A few lines
of local code often beat a dependency.

### 1.4 Where it is enforced

- The **work item** carries a `New dependencies` section when the work needs one, naming
  each package exactly as the registry spells it, with its manifest and the date of the
  approval. A work item that does not name a package does not approve it.
- The **review** lists every new direct dependency in the diff and flags each one the work
  item does not name ([pr review, 21-agent-working-rules](21-agent-working-rules.md)).
  For a package manifest, compare the **key sets** at the base and the head, not the added
  lines: a re-sorted block shows added lines for keys that are not new.
- The **pull request body** records, for each new package, the six checks of § 1.3 and
  why existing code does not do the job.

---

## 2. Install scripts are off everywhere

A package manager runs each dependency's pre-install, install and post-install scripts
during an install, with the rights of the user. Most package worms start from a
post-install script. On a build box such a script reads the package-manager credentials,
the pipeline tokens, the test credentials and the cached cloud sessions. **A lock file
does not stop this when the locked version is the bad one.**

- **Turn scripts off in a committed per-project configuration file**, not with a flag on
  each install call. A flag has to be remembered at fifty call sites and in every agent
  session. A committed file reaches every machine through a `git pull`, needs no new
  dependency, and changes no user-level or global setting.
- **One allowlist file** names, per project, every dependency that has an install script
  in the lock file, as `run` or `skip`, with a reason that says what the script does.
- **A separate command runs the allowed scripts on purpose**, after the install, in the
  pipelines that need them. Note the trap: rebuilding a package by name does **not** run
  its script while scripts are off; the command must pass the override flag explicitly.
- **A guard test** fails when a lock file and the allowlist disagree, when a project is
  missing the configuration file or does not set the option, and when a project's own
  scripts rely on a hook the setting now skips.
- **An agent never turns scripts back on to make a package work.** It adds the package to
  the allowlist with a reason instead.

**Known consequence, and it will bite on adoption day:** turning scripts off also stops
the project's **own** pre- and post- hooks and its prepare script. Fold each one into the
first command of the script it belonged to, and have the guard test refuse a new hook.

---

## 3. A new version waits

A compromised release is usually found and pulled within days. Waiting removes most of
the exposure for free.

- **Fail a change that adds a package version younger than 7 days.** The check reads the
  changed lock files and asks the registry for each changed version's publish time.
- **One override, and only the owner sets it.** Reference A uses a label on the pull
  request, which prints the list and passes. An agent never sets it.
- **The automated dependency updater waits too.** Set its cooldown to the same 7 days, in
  one group, weekly. The best-known moved-tag incident was found and pulled inside that
  window.

---

## 4. Advisories, and where a package came from

- **Fail a change that ADDS a high or critical advisory.** Advisories the base commit
  already carries are **listed and do not fail the check** — otherwise the gate is red
  for everyone until an unrelated backlog is cleared, and a red gate that nobody can turn
  green teaches people to ignore it.
- **Fail a lock file entry resolved from a host other than the package registry.** A
  dependency pointing at an arbitrary host is a second supply chain nobody reviewed.
- **The check runs no install.** Audit from the lock file only. An install in the security
  check would run the thing the check exists to judge.

---

## 5. Pipeline actions are pinned to a commit

A tag is a pointer. The owner of an action — or anyone who takes over that account — can
move it to other code. In one well-known incident the tags of a widely used action were
moved to a commit that printed runner secrets into the build log, and every workflow that
named a tag ran it.

This matters most where the pipeline holds the deploy role: a moved tag in a deploy
workflow runs code with the permissions of that role.

- **Each remote action reference is `owner/repo@<40-hex commit SHA> # vX.Y.Z`.** The
  comment names the tag the SHA came from. The automated updater reads the comment and
  writes the new SHA and the new comment together.
- **A reference to an action inside the same repository needs no SHA**: it runs from the
  same commit.
- **The first pins change no behaviour.** Take the commit each existing tag resolves to
  today.
- **A guard test** fails on any remote reference that is not a 40-character SHA with a
  version comment, and pins the updater's settings.
- **Run that test on pull requests.** A test that runs only on the integration branch
  lets a tag merge and run before the test reads it.
- **Known cost:** a major-version bump arrives in the same weekly pull request as the
  patch bumps, and a major bump can change inputs or the runtime. That pull request
  sometimes needs edits before it merges.

---

## 6. Container and binary inputs are pinned by digest or hash

Packages are not the only input. A base image tag moves, and a download URL serves
whatever is behind it today.

- **Name the base image by digest**, not by tag.
- **Compile the dependency list with hashes.** Keep the top-level packages in one file,
  and generate the locked file from it with hashes for **every** package, transitive ones
  included. Then install with the flags that make the hashes load-bearing: require
  hashes, no implicit dependency resolution, binary artifacts only.
- **Check every downloaded binary by SHA-256**, from a versioned URL, with a bounded
  retry. Print the file label and both hashes on a mismatch.
- **A guard test** fails on a requirements line with no hash, an install without the
  require-hashes flag, a `FROM` without a digest, and a download with no hash check.
- **That test must RUN the check, not read the script as text.** A text-reading test stays
  green when someone inverts the comparison inside the validator. Run each validator in a
  shell against a well-formed file of the right format whose hash is wrong, and assert it
  refuses; then assert it accepts the one file whose hash matches
  ([11-testing-unit](11-testing-unit.md), mutation discipline).

---

## 7. The pipeline holds no long-lived credential

- **Federate.** The pipeline signs in to the cloud with short-lived credentials issued per
  run. No long-lived key is stored in the pipeline's secrets.
- **The trusted subject must carry the branch.** A subject that names only the deployment
  environment is satisfied by **any** branch that can run a workflow naming that
  environment. Customise the subject to include the ref, and list the branches each role
  accepts in the trust policy — in the cloud, where a change to a pipeline setting cannot
  widen it.
- **A plan is a read-only action and gets a read-only role.** Give it explicit denials on
  writing the state and on taking the state lock. The planning step runs with no lock and
  **never falls back to the write role**; it stops when the read-only role is not
  configured.
- **A production write role trusts the release branch only.** Watch for the trap: a run
  started by another workflow's completion carries the default branch as its ref, so a
  production step started that way cannot assume a release-branch-only role. Make it a
  called job inside the workflow that runs on the release branch.
- **Say out loud what the read-only role can still read.** A broad read-only policy can
  read every object and every table in the account, which includes member data, and any
  branch can run a plan.

---

## 8. Generated artifacts are not committed

Contract artifacts, clients and route tables are generated on every build and never
committed ([06-api-contract-and-codegen](06-api-contract-and-codegen.md)). That is a
supply-chain control as well as a correctness one: a committed generated file is a file a
reviewer skims, and some of them are read by the running service to make an authorization
decision. **The review surface is the declaration the file is generated from**, not the
file.

---

## 9. Where each check runs

| Check | Event | Fails when |
|---|---|---|
| Dependency named or approved (§ 1) | Review, by a person | A new manifest entry is in neither the work item nor an approval |
| Install-script allowlist (§ 2) | Unit suite | A lock file and the allowlist disagree, or a project lost the setting |
| Release age (§ 3) | Pull request, **and** push to the integration and release branches | A changed version is younger than 7 days and no override label is set |
| Advisories and package source (§ 4) | Pull request, **and** push to the integration and release branches | The change adds a high or critical advisory, or a non-registry entry |
| Action pins (§ 5) | Pull request | A remote reference is not a SHA with a version comment |
| Build-input pins (§ 6) | Unit suite | A requirement has no hash, an image has no digest, a download has no hash check |
| Role trust (§ 7) | Infrastructure guard test | A role trusts a subject without a branch, or the plan role can write |

Two notes on the table, both learned the hard way:

- **Run the supply-chain checks on a push to the integration and release branches as
  well**, with the push's own previous commit as the base. Otherwise a direct push is the
  way round them. A push has no override.
- **Cancel a superseded pull-request run; never cancel a push run.** A pull-request run
  describes a commit that is no longer current. A push run checks only its own commit
  range, and a cancelled one leaves that range unchecked forever.

---

## 10. Adopting it

1. Write § 1 into the repository's rules file and into the review checklist. It costs
   nothing and closes the invented-package path today.
2. Turn install scripts off, build the allowlist from the current lock files, and fold the
   hooks the setting breaks.
3. Add the release-age check and the advisory check, on pull requests and on pushes.
4. Pin every pipeline action to a SHA, turn on the automated updater with a 7-day cooldown,
   and add the guard test to the pull-request checks.
5. Pin the container and binary inputs, and write the guard test that **runs** each
   validator.
6. Move the pipeline to short-lived federated credentials, put the branch in the trusted
   subject, and split the plan role from the apply role.
