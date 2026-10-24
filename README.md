# snp

`snp` is the command-line tool of Sinopia, shared memory for software agents. It is alpha, no SLA.

## Install

On Linux (amd64 or arm64), with `curl`, `openssl` 3.0 or later and `sha256sum`:

```
curl -fsSL https://github.com/Tesselabs/snp/releases/latest/download/snp-install.sh | sh
```

`snp-install.sh` verifies the signed release before it installs anything: the key statement
against the recovery key, the release manifest against a release key, and every file against
the manifest. It asks the Sinopia server which version to install, puts `snp` in
`/usr/local/bin` when that is yours (or root's) and writable, else in `~/.local/bin`, and
installs its skills into `~/.claude` and `~/.agents/skills`. Run it again to reinstall. A
download cut short runs nothing: the script's last line starts the work. To read it before it
runs, download it with `curl -fsSLO` and the same address, then run `sh snp-install.sh`.

On a terminal it then asks for your org, unless `SNP_ORG` is set or `~/.snp/config` already
names one, and saves the answer in `~/.snp/config`.

Unless `~/.snp/no-auto-update` exists or `~/.snp` is not safe (run `snp skills install --user`
to see why), the install also adds a SessionStart hook to `~/.claude/settings.json`, so each new
Claude Code session runs `snp update --auto` and picks up a newer `snp`. `snp` rewrites only `hooks.SessionStart` there; key order and spacing may change.
Turn the hook off with `snp skills install --user --no-auto-update`
and back on with `--auto-update`. Builds that are not released write no hook.
To remove what `--user` installed, run `snp skills uninstall --user`: it deletes the `snp-*`
skills and agents that install listed in `~/.snp/skills-installed` and the SessionStart hook, and
leaves your other skills and agents alone. `settings.json` may be reformatted, and an empty
`"hooks": {}` can remain. It also leaves `~/.snp/no-skills`: while that file is there, `snp update`
still installs a newer `snp`, but no skills and no hook. `snp skills install --user` removes it.

## Configure

`snp` talks to the Sinopia server at `https://snp.tesselabs.com`. It needs two values from you,
your org and a token; ask whoever runs the server for both. The org goes in `~/.snp/config`
(`snp-install.sh` asks for it), the token in `SNP_TOKEN`:

```
mkdir -p -m 700 ~/.snp
printf 'org=%s\n' your-org >> ~/.snp/config
export SNP_TOKEN=your-token
snp mem list --limit 1
```

Keep the token out of shell history and out of files in a repository.

`SNP_ORG` overrides the file's `org=`. To use another server, add `url=` with its address to
`~/.snp/config`, or set `SNP_URL`, which also wins over the file and which `snp-install.sh`
reads too.

### claude.ai cloud environments

In the environment settings at claude.ai/code:

- **Network access:** Custom, keep the defaults, and add `snp.tesselabs.com`.
- **Network secret:** type Bearer for `snp.tesselabs.com`, header `Authorization`, prefix
  `Bearer`, value: your token. The session's proxy adds it to each request, so `SNP_TOKEN`
  stays unset and the token never enters the session.
- **Setup script:** the install, with your org first:

  ```
  mkdir -p -m 700 ~/.snp
  printf 'org=%s\n' your-org > ~/.snp/config
  curl -fsSL https://github.com/Tesselabs/snp/releases/latest/download/snp-install.sh | sh
  ```

  It installs `snp` and its skills into `~/.claude` and `~/.agents/skills` for every session.
  For another server, add `export SNP_URL=` with its address before the install.

To keep the installer in the setup script itself instead of downloading it, paste
`snp-install.sh` whole in place of the `curl` line; it is the block below, byte for byte.

<details>
<summary>snp-install.sh</summary>

```sh
#!/bin/sh
# Sinopia snp installer paste. It checks signed release metadata before it
# installs anything. Inputs, all optional: SNP_URL, SNP_RELEASE_BASE,
# SNP_ASSET_DIR with SNP_TAG.
set -u
LC_ALL=C
export LC_ALL
# Only the command line configures curl (-q, first on every call) and openssl.
unset OPENSSL_CONF OPENSSL_MODULES

SNP_MIN_VERSION=v0.2.0
SNP_MIN_KEYS_SEQ=1
SNP_RECOVERY_KEY=1U/IvEldYgf2P1yKx2DlX+tuvR4fHsQmwqgV++Ynoyw=
SNP_RELEASE_KEYS="581BCDWgLCK2876qn/HCFHB1pI3AtvezbTdO+PzR1Ic="

base=${SNP_RELEASE_BASE:-https://github.com/Tesselabs/snp/releases}
# The built-in server goes with the built-in release host only: a release from
# SNP_RELEASE_BASE or SNP_ASSET_DIR asks no server unless SNP_URL names one.
if [ -z "${SNP_RELEASE_BASE:-}" ] && [ -z "${SNP_ASSET_DIR:-}" ]; then
	SNP_URL=${SNP_URL:-https://snp.tesselabs.com}
fi
keys_floor=$SNP_MIN_KEYS_SEQ
version_floor=$SNP_MIN_VERSION
tmp=
arch=
tag=
latest=

# fail prints the one stderr line on fd 3: main points fd 2 at /dev/null so
# that no tool can add a second line.
fail() {
	case $1 in
	openssl) m='OpenSSL 3.0 or later is required to check signatures; install it and run the paste again' ;;
	net) m='a download failed; check the connection and run the paste again' ;;
	keys) m='the signed key statement is missing or older than this installer accepts, so nothing was installed; run the paste again later, and if it fails again, send this line to whoever gave you the paste' ;;
	sig) m='a signature did not verify, so nothing was installed; do not retry: send this line to whoever gave you the paste' ;;
	key) m='the release is signed by an unknown or revoked key, so nothing was installed; do not retry: send this line to whoever gave you the paste' ;;
	parse) m='the release metadata is malformed, so nothing was installed; do not retry: send this line to whoever gave you the paste' ;;
	floor) m='the release is older than the minimum version, so nothing was installed; run the paste again later, and if it fails again, send this line to whoever gave you the paste' ;;
	version) m='the signed version does not match the release tag, so nothing was installed; run the paste again later, and if it fails again, send this line to whoever gave you the paste' ;;
	sum) m='a downloaded file does not match its signed sha256, so nothing was installed; do not retry: send this line to whoever gave you the paste' ;;
	install) m='snp could not be installed; it needs Linux on amd64 or arm64, a writable /usr/local/bin or ~/.local/bin, and a writable ~/.claude and ~/.agents; fix that and run the paste again' ;;
	esac
	printf 'snp-install: %s: %s\n' "$1" "$m" >&3
	exit 1
}

# matches <ere> <string>: the whole string is one line and matches the ERE.
matches() {
	case $2 in *'
'*) return 1 ;; esac
	printf '%s\n' "$2" | grep -Eq "$1"
}

# less <a> <b>: a < b, for digit strings or vX.Y.Z versions compared part by
# part. Parts compare as digit strings, so no value overflows.
less() {
	awk -v a="$1" -v b="$2" '
	function n(s) { sub(/^0+/, "", s); return s == "" ? "0" : s }
	function lt(x, y) {
		x = n(x); y = n(y)
		if (length(x) != length(y)) return length(x) < length(y)
		return ("" x) < ("" y)
	}
	BEGIN {
		sub(/^v/, "", a); sub(/^v/, "", b)
		k = split(a, A, "."); split(b, B, ".")
		for (i = 1; i <= k; i++) {
			if (lt(A[i], B[i])) exit 0
			if (lt(B[i], A[i])) exit 1
		}
		exit 1
	}'
}

# keyid <base64 public key>: the first 8 hex of its sha256.
keyid() {
	printf '%s' "$1" | base64 -d 2>/dev/null | sha256sum | cut -c1-8
}

# verify <base64 public key> <file> <sig file>: an Ed25519 signature check that
# passes only on exit 0 and stdout of exactly the success line and one LF.
verify() {
	der=$tmp/key.der
	{
		printf '\060\052\060\005\006\003\053\145\160\003\041\000'
		printf '%s' "$1" | base64 -d 2>/dev/null
	} >"$der" || return 1
	out=$tmp/verify.out
	openssl pkeyutl -verify -pubin -inkey "$der" -keyform DER -rawin \
		-in "$2" -sigfile "$3" >"$out" 2>/dev/null || return 1
	[ "$(wc -c <"$out" | tr -d ' ')" = 32 ] &&
		[ "$(cat "$out")" = 'Signature Verified Successfully' ]
}

# text_ok <file>: not empty, no NUL byte, and the last byte is LF. The awk
# parsers below check the rest of the shape.
text_ok() {
	[ -s "$1" ] || return 1
	[ "$(tail -c 1 "$1" | od -An -tx1 | tr -d ' \n')" = 0a ] || return 1
	! od -An -v -tx1 "$1" | grep -q ' 00'
}

# Every awk comparison of an id, sha, key or version concatenates "" so it is a
# string comparison: awk compares numeric-looking strings as numbers, and
# 00123400 would equal 001234e2.
#
# Shared awk functions for the strict parsers. They mirror
# contract.ParseManifest and contract.ParseKeys.
awk_lib='
function hex(s, n) { return length(s) == n && s !~ /[^0-9a-f]/ }
function digits(s) { return s != "" && s !~ /[^0-9]/ }
function seqok(l,   s) {
	if (substr(l, 1, 4) != "seq ") return 0
	s = substr(l, 5)
	if (!digits(s) || s ~ /^0/) return 0
	return length(s) < 20 || (length(s) == 20 && s <= "18446744073709551615")
}
function issuedok(l,   d, y, mo, dd, dim) {
	if (length(l) != 27 || substr(l, 1, 7) != "issued ") return 0
	d = substr(l, 8)
	if (d !~ /^[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]T[0-9][0-9]:[0-9][0-9]:[0-9][0-9]Z$/) return 0
	y = substr(d, 1, 4) + 0; mo = substr(d, 6, 2) + 0; dd = substr(d, 9, 2) + 0
	if (mo < 1 || mo > 12 || dd < 1) return 0
	if (substr(d, 12, 2) + 0 > 23 || substr(d, 15, 2) + 0 > 59 || substr(d, 18, 2) + 0 > 59) return 0
	dim = substr("312831303130313130313031", mo * 2 - 1, 2) + 0
	if (mo == 2 && y % 4 == 0 && (y % 100 != 0 || y % 400 == 0)) dim = 29
	return dd <= dim
}
function part(s) { return s == "0" || (digits(s) && s !~ /^0/) }
function versionok(l,   p) {
	if (substr(l, 1, 9) != "version v") return 0
	return split(substr(l, 10), p, ".") == 3 && part(p[1]) && part(p[2]) && part(p[3])
}
function b64ok(s) {
	return length(s) == 44 && substr(s, 44, 1) == "=" && substr(s, 1, 43) !~ /[^A-Za-z0-9+\/]/ &&
		index("AEIMQUYcgkosw048", substr(s, 43, 1)) > 0
}
/\r/ { bad = 1 }
'

# parse_keys <file>: checks a snp-keys statement and prints "seq <n>", then
# its "release <id> <base64>" and "revoke <id>" lines. The caller checks that
# each id is the key-id of its key.
parse_keys() {
	text_ok "$1" || return 1
	awk "$awk_lib"'
	NR == 1 { if ($0 != "snp-keys v1") bad = 1; next }
	NR == 2 { if (seqok($0)) seq = substr($0, 5); else bad = 1; next }
	NR == 3 { if (!issuedok($0)) bad = 1; next }
	substr($0, 1, 8) == "release " {
		id = substr($0, 9, 8); k = substr($0, 18)
		if (length($0) != 61 || substr($0, 17, 1) != " " || !hex(id, 8) || !b64ok(k) || nrev || (id in rel)) bad = 1
		rel[id] = 1; nrel++; out = out "release " id " " k "\n"; next
	}
	substr($0, 1, 7) == "revoke " {
		id = substr($0, 8)
		if (!hex(id, 8) || (id in rel) || (id in rev)) bad = 1
		rev[id] = 1; nrev++; out = out "revoke " id "\n"; next
	}
	{ bad = 1 }
	END {
		if (bad || NR < 4 || !nrel) exit 1
		printf "seq %s\n%s", seq, out
	}' "$1"
}

# parse_manifest <file>: checks a snp-release file.
parse_manifest() {
	text_ok "$1" || return 1
	awk "$awk_lib"'
	BEGIN { split("LICENSE NOTICE install.sh snp-linux-amd64 snp-linux-arm64", want, " ") }
	NR == 1 { if ($0 != "snp-release v1") bad = 1; next }
	NR == 2 { if (substr($0, 1, 7) != "key-id " || !hex(substr($0, 8), 8)) bad = 1; next }
	NR == 3 { if (!versionok($0)) bad = 1; next }
	NR == 4 { if (!seqok($0)) bad = 1; next }
	NR == 5 { if (!issuedok($0)) bad = 1; next }
	NR <= 10 { if (!hex(substr($0, 1, 64), 64) || substr($0, 65) != "  " want[NR - 5]) bad = 1; next }
	{ bad = 1 }
	END { if (bad || NR != 10) exit 1 }' "$1"
}

check_openssl() {
	v=$(openssl version 2>/dev/null) || fail openssl
	matches '^OpenSSL ([3-9]|[1-9][0-9]+)\.' "$v" || fail openssl
}

# detect_arch sets arch to the release arch name of this machine.
detect_arch() {
	[ "$(uname -s 2>/dev/null)" = Linux ] || fail install
	case $(uname -m 2>/dev/null) in
	x86_64) arch=amd64 ;;
	aarch64 | arm64) arch=arm64 ;;
	*) fail install ;;
	esac
}

# healthz reads /healthz (only when there is an SNP_URL) and raises keys_floor and
# version_floor to what it reports; latest_client becomes the target. A value
# that is not a plain version or number is ignored and never printed, and any
# failure leaves the inlined floors.
healthz() {
	[ -n "${SNP_URL:-}" ] || return 0
	body=$(curl -q -fsS --proto =http,https --max-time 10 "$SNP_URL/healthz" 2>/dev/null | head -c 65536) || return 0
	v=$(printf '%s\n' "$body" | sed -n 's/.*"min_client_version" *: *"\([^"]*\)".*/\1/p')
	if matches '^v[0-9]+\.[0-9]+\.[0-9]+$' "$v" && less "$version_floor" "$v"; then
		version_floor=$v
	fi
	n=$(printf '%s\n' "$body" | sed -n 's/.*"min_keys_seq" *: *\([^ ,}]*\).*/\1/p')
	if matches '^[1-9][0-9]*$' "$n" && less "$keys_floor" "$n"; then
		keys_floor=$n
	fi
	v=$(printf '%s\n' "$body" | sed -n 's/.*"latest_client" *: *"\([^"]*\)".*/\1/p')
	if matches '^v[0-9]+\.[0-9]+\.[0-9]+$' "$v"; then
		latest=$v
	fi
	return 0
}

# find_tag sets tag, the release tag to install: SNP_TAG with SNP_ASSET_DIR,
# else /healthz's latest_client, else where /latest redirects to.
find_tag() {
	if [ -n "${SNP_ASSET_DIR:-}" ]; then
		tag=${SNP_TAG:-}
		matches '^v[0-9]+\.[0-9]+\.[0-9]+$' "$tag" || fail version
		return 0
	fi
	if [ -n "$latest" ]; then
		tag=$latest
		return 0
	fi
	loc=$(curl -q -sS --proto =http,https --max-time 30 -o /dev/null \
		-w '%{redirect_url}' "$base/latest" 2>/dev/null) || fail net
	tag=${loc##*/}
	matches '^v[0-9]+\.[0-9]+\.[0-9]+$' "$tag" || fail net
}

# fetch <file>: gets one release file into the temp dir, from SNP_ASSET_DIR
# when set (a missing file is 22, like an HTTP error) or by download. It
# returns curl's exit status.
fetch() {
	if [ -n "${SNP_ASSET_DIR:-}" ]; then
		[ -f "$SNP_ASSET_DIR/$1" ] || return 22
		cp "$SNP_ASSET_DIR/$1" "$tmp/$1" 2>/dev/null
		return $?
	fi
	curl -q -fsSL --proto =http,https --max-time 300 -o "$tmp/$1" \
		"$base/download/$tag/$1" 2>/dev/null
}

# download_all fetches the key statement, the manifest and the files this arch
# installs. A missing key statement is keys; every other failure is net.
download_all() {
	for f in snp-keys snp-keys.sig; do
		rc=0
		fetch "$f" || rc=$?
		[ "$rc" -eq 0 ] && continue
		[ "$rc" -eq 22 ] && fail keys
		fail net
	done
	for f in snp-release snp-release.sig LICENSE NOTICE install.sh "snp-linux-$arch"; do
		fetch "$f" || fail net
	done
}

# trusted_keys checks snp-keys (rules 2 to 4) and writes the trusted release
# keys to $tmp/trusted as "<id> <base64>" lines.
trusted_keys() {
	verify "$SNP_RECOVERY_KEY" "$tmp/snp-keys" "$tmp/snp-keys.sig" || fail sig
	parse_keys "$tmp/snp-keys" >"$tmp/keys.parsed" || fail parse
	while read -r kind id k; do
		[ "$kind" = release ] || continue
		[ "$(keyid "$k")" = "$id" ] || fail parse
	done <"$tmp/keys.parsed"
	seq=$(sed -n '1s/^seq //p' "$tmp/keys.parsed")
	less "$seq" "$keys_floor" && fail keys
	for k in $SNP_RELEASE_KEYS; do
		printf 'release %s %s\n' "$(keyid "$k")" "$k"
	done >"$tmp/keys.all"
	sed 1d "$tmp/keys.parsed" >>"$tmp/keys.all"
	awk -v rec="$SNP_RECOVERY_KEY" '
	$1 == "release" && ($3 "") != (rec "") { k[$2 ""] = $3 }
	$1 == "revoke" { r[$2 ""] = 1 }
	END { for (id in k) if (!(id in r)) print id, k[id] }' "$tmp/keys.all" >"$tmp/trusted"
}

# check_manifest verifies snp-release (rules 5 to 7) and sets version.
check_manifest() {
	line=$(sed -n 2p "$tmp/snp-release")
	matches '^key-id [0-9a-f]{8}$' "$line" || fail parse
	k=$(awk -v id="${line#key-id }" '($1 "") == (id "") { print $2; exit }' "$tmp/trusted")
	[ -n "$k" ] || fail key
	verify "$k" "$tmp/snp-release" "$tmp/snp-release.sig" || fail sig
	parse_manifest "$tmp/snp-release" || fail parse
	version=$(sed -n '3s/^version //p' "$tmp/snp-release")
	less "$version" "$version_floor" && fail floor
	[ "$version" = "$tag" ] || fail version
	# Rule 7 in every mode: with SNP_ASSET_DIR the tag is SNP_TAG, so check
	# latest_client too.
	[ -z "$latest" ] || [ "$version" = "$latest" ] || fail version
}

# check_sums compares each file this arch installs with its signed sha256.
check_sums() {
	for f in LICENSE NOTICE install.sh "snp-linux-$arch"; do
		want=$(awk -v f="$f" 'NR > 5 && ($2 "") == (f "") { print $1 }' "$tmp/snp-release")
		got=$(sha256sum "$tmp/$f" 2>/dev/null | cut -d' ' -f1)
		if [ -z "$got" ] || [ "$got" != "$want" ]; then
			fail sum
		fi
	done
}

main() {
	exec 3>&2 2>/dev/null
	check_openssl
	detect_arch
	tmp=$(mktemp -d 2>/dev/null) || fail install
	trap 'rm -rf "$tmp"' EXIT
	trap 'exit 1' HUP INT TERM
	healthz
	find_tag
	download_all
	trusted_keys
	check_manifest
	check_sums
	sh "$tmp/install.sh" "$tmp/snp-linux-$arch" "$version" </dev/null 2>&3
}

main "$@"
```

</details>

## Key fingerprints

- Recovery key: `1U/IvEldYgf2P1yKx2DlX+tuvR4fHsQmwqgV++Ynoyw=` (key id `7c39f0ce`)
- Release key: `581BCDWgLCK2876qn/HCFHB1pI3AtvezbTdO+PzR1Ic=` (key id `cfaf473a`)

Each is the base64 Ed25519 public key; the key id is the first 8 hex digits of its sha256.
The release key is the genesis key the paste embeds. A `snp-keys` statement signed by the
recovery key can rotate it: each release ships its statement, and the paste trusts only the
release keys that statement allows.

## Licence

Copyright 2026 Tesselabs LLC. See [LICENSE](LICENSE) for the terms and [NOTICE](NOTICE) for third-party notices.
