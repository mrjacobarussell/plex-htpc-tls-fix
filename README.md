# Plex HTPC (Linux snap) — "certificate verify failed" playback fix

## Symptom

Plex HTPC (`snap install plex-htpc`) on Linux fails **every** playback attempt with:

```
ffmpeg: tls: error:0A000086:SSL routines::certificate verify failed
stream: Failed to open https://<host>.<hash>.plex.direct:32400/...
```

The app otherwise works fine — you can log in and browse your library, playback just never starts. Error shown in the UI is a generic "An unexpected playback problem occurred."

## Root cause

This is **not** a stale-package problem in the way it first looks. Plex's `plex.direct` dynamic-DNS certificates are issued by Let's Encrypt from a newly-generated certificate chain:

- Intermediate: `Let's Encrypt YR2`
- Root: **`ISRG Root YR`** (generated 2025-09-03)

As of this writing, `ISRG Root YR` is **not yet included in any standard trust store** — not Mozilla's bundle, not Ubuntu's `ca-certificates` package, nothing. Confirmed by testing against a fully up-to-date Ubuntu system:

```bash
curl -sv https://<your-server-ip-dashed>.<hash>.plex.direct:32400/identity
# SSL certificate OpenSSL verify result: unable to get local issuer certificate (20)
```

This fails identically on the live system CA store, not just inside the snap.

On top of that, Plex HTPC's own code **hardcodes** its TLS CA file path to the snap's bundled `resources/cacert.pem` via an internal mpv property set — it ignores `SSL_CERT_FILE`/`SSL_CERT_DIR` environment variables, and overrides anything set in a user `mpv.conf` a few hundred milliseconds after it loads. So even fixing your system trust store alone does **not** fix the app — you have to patch the bundle the app actually reads.

Reference: Let's Encrypt's own certificate page (https://letsencrypt.org/certs/) lists `ISRG Root YR` under **"New Roots (Not Yet in Trust Stores)"**.

## The fix

### 1. Add the new root to your system (harmless, fixes other apps too)

```bash
curl -s https://letsencrypt.org/certs/gen-y/root-yr.pem -o /tmp/root-yr.pem
sudo cp /tmp/root-yr.pem /usr/local/share/ca-certificates/isrg-root-yr.crt
sudo update-ca-certificates
```

Verify:

```bash
openssl x509 -in /tmp/root-yr.pem -noout -subject -dates
# subject=C = US, O = ISRG, CN = Root YR
```

### 2. Patch the Plex HTPC snap's bundled cert file

The snap ships as a read-only squashfs mount, and the app ignores the system trust store, so you need to rebuild the package with a patched bundle.

```bash
# Find the installed snap's revision and backing file
snap list plex-htpc
# e.g. plex-htpc  1.71.1  73  latest/stable  plexinc**

sudo mkdir -p /tmp/plex-htpc-repack
cd /tmp/plex-htpc-repack
sudo unsquashfs -d extracted /var/lib/snapd/snaps/plex-htpc_73.snap   # use your revision number

# Replace the stale bundle with your current system one, then add the new root
sudo cp /etc/ssl/certs/ca-certificates.crt extracted/resources/cacert.pem
sudo bash -c 'cat /tmp/root-yr.pem >> extracted/resources/cacert.pem'

# Repack
sudo mksquashfs extracted plex-htpc-patched.snap -comp xz -no-fragments -no-progress

# Install locally (bypasses signature check since we modified the package)
sudo snap install plex-htpc-patched.snap --dangerous
```

This installs as a new local revision (e.g. `x1`) alongside/instead of the official one. `snap list plex-htpc` will show it with no publisher listed, confirming it's the locally-patched build.

### 3. Verify the actual fix before relying on it

```bash
echo | openssl s_client -connect <your-server-ip-dashed>.<hash>.plex.direct:32400 \
  -servername <your-server-ip-dashed>.<hash>.plex.direct \
  -CAfile extracted/resources/cacert.pem 2>&1 | grep "Verify return code"
# Verify return code: 0 (ok)
```

Then relaunch Plex HTPC and try playback.

## Caveats

- **This is a local workaround, not a redistribution.** This repo contains no Plex binaries, no decompiled code, and no copies of the snap package — only the shell commands to patch the certificate bundle on your own already-installed copy. You need your own legitimate `snap install plex-htpc` first.
- If `snap refresh` ever pulls a new official revision, it will overwrite this patch and the bug may return until Plex ships their own fix with a caught-up CA bundle (which should happen naturally as `ISRG Root YR` propagates into standard distro `ca-certificates` packages over time).
- Check `snap list plex-htpc` after any snap refresh — if the revision number changed, re-run step 2.

## Tested on

- Ubuntu Server 24.04 LTS, Plex HTPC 1.71.1 (snap rev 73), Intel UHD 630 iGPU.
