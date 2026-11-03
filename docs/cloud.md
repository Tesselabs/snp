# claude.ai cloud environments

A claude.ai cloud session gets `snp` from its environment. The setup script below runs `snp-install.sh` from the latest release, which verifies the signed release before it installs anything. Every session the environment opens then has `snp`, its skills in `~/.claude` and `~/.agents/skills`, and the SessionStart hook that runs `snp update --auto`. A new release needs no change in the environment.

You need your org and a token; ask whoever runs the Sinopia server for both.

## Set up an environment

At claude.ai/code, create an environment with the network access and setup script below and save it, then add the network secret:

| Field | Value |
|---|---|
| Network access | **Custom**, keep the defaults, add the Sinopia server (`snp.tesselabs.com`) |
| Network secret | Name `snp`, type Bearer, Allowed websites: the Sinopia server (`snp.tesselabs.com`), header `Authorization`, prefix `Bearer`, value: your token |
| Setup script | The block below, with your org |

```bash
mkdir -p -m 700 ~/.snp
printf 'org=%s\n' your-org > ~/.snp/config
curl -fsSL https://github.com/Tesselabs/snp/releases/latest/download/snp-install.sh -o ~/.snp/snp-install.sh && sh ~/.snp/snp-install.sh || echo "snp NOT installed: see the line above"
```

The session's proxy adds the token to each request, so `SNP_TOKEN` stays unset and the token never enters the session. To check the setup, run `snp mem list --limit 1` in a session.

The install downloads from GitHub, so keep the defaults. The Sinopia server goes in the allowed domains because setup-script requests do not get the network secret: the proxy adds it only after setup ends. If setup gets no release name from the server, the install takes the release GitHub marks as latest.

To use another server, use its host in the network access and the network secret, add `printf 'url=%s\n' https://your-host >> ~/.snp/config` after the org line, and put `SNP_URL=https://your-host` before `sh ~/.snp/snp-install.sh`.

## When the install fails

The setup log shows the failure. A failed download prints a `curl:` line and a failed install prints a `snp-install: <code>: ...` line. Either way `snp NOT installed: see the line above` follows and setup finishes. The session then has no `snp` command and no Sinopia skills, with one exception: `snp-install: install: snp was installed, ...`. That line comes after `snp` is in place, so the session has `snp` even though `snp NOT installed` follows. The next line is the error from `snp`: the Sinopia skills, the hook or both can be missing. The Sinopia block downloads first because `curl ... | sh` hides a failed `curl`: `sh` reads nothing and exits 0.

Nothing in a session retries a failed install. Running the Sinopia block again in a session fixes that session only: a session's writes are lost, so the environment's snapshot stays as setup left it. After a `curl:` line, or any `snp-install:` line other than `snp was installed`, the snapshot has no `snp`, Sinopia skills or hook. After `snp was installed`, the snapshot has `snp`, and the Sinopia skills, the hook or both can be missing. The next install attempt is a rebuild, which runs the Sinopia block again.

An edit to the environment's setup script rebuilds it, and so does the snapshot's expiry after about 7 days. The failure lines call `snp-install.sh` "the paste":

- If the line says `do not retry`, stop and send it to whoever runs the Sinopia server.
- If it says to run the paste again, fix the cause it names, if it names one, then edit the setup script.
- If it is a `curl:` line or says `snp was installed`, fix the cause, then edit the setup script. The `snp skills install --user` that the `snp was installed` line names fixes that session only.

## Add Superpowers

Superpowers is a separate set of agent skills. Sinopia does not ship it or install it. If a Sinopia skill stops because Superpowers is missing, add this block to the environment's setup script, after the Sinopia block:

```bash
git clone -q https://github.com/obra/superpowers.git /tmp/sp && git -C /tmp/sp checkout -q bb92a77741419a4ab5f06e711a283343f1ada0c3 && mkdir -p ~/.claude/skills && cp -r /tmp/sp/skills/. ~/.claude/skills/ && echo "superpowers skills installed" || echo "superpowers NOT installed"
```

The line copies the skills of one fixed Superpowers commit into `~/.claude/skills`. It downloads from GitHub, which the defaults you kept allow. The edit rebuilds the environment, and the Sinopia block runs again.

A failed line prints `superpowers NOT installed` and setup still finishes. The blocks fail separately: a failure in one does not stop or change the other. Sessions can lack some or all Superpowers skills until a later rebuild succeeds.
