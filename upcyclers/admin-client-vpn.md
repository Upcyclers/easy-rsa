# Upcyclers Admin Client VPN PKI Runbook

This EasyRSA fork is used as the tool source for managing Upcyclers admin AWS
Client VPN certificates. It is not the source of truth for generated VPN
credentials.

Do not commit generated PKI state, `.ovpn` files, private keys, certificate
revocation lists, or encrypted PKI archives to Git.

## Responsibility

The associated admin technical owner, CTO, or delegated infrastructure developer
is responsible for admin VPN certificate operations.

That owner is responsible for:

- Maintaining the encrypted EasyRSA `pki/` archive in 1Password.
- Generating one client certificate per team member or client.
- Creating personalized `.ovpn` profiles.
- Sending `.ovpn` files only through approved secure channels.
- Revoking certificates when access should be removed.
- Importing updated CRLs into AWS Client VPN.

Clients and regular team members should not run EasyRSA. They should only
install AWS VPN Client, import the provided `.ovpn` profile, connect, and open
the admin dashboard.

## Storage Model

This repository stores only EasyRSA source code and Upcyclers operating
instructions.

Sensitive state lives in 1Password:

```text
1Password item: Admin Client VPN PKI
```

The 1Password item should store encrypted PKI archives, for example:

```text
admin-stage-client-vpn-pki.zip
admin-prod-client-vpn-pki.zip
```

The generated `pki/` directory is the actual certificate authority state. It
contains the CA key, issued certificates, client private keys, serial/index
database, and CRL state. Losing it makes per-user issuance and revocation
difficult; leaking it compromises VPN access.

The upstream EasyRSA `.gitignore` already ignores:

```text
easyrsa3/pki
```

Do not bypass that ignore rule.

## Restoring PKI State

To issue or revoke VPN certificates, restore the correct environment PKI archive
from 1Password:

```bash
DOWNLOADED_PKI_ZIP="<absolute_path_to_downloaded_pki_zip>"
cd "<easy_rsa_fork>/easyrsa3"
test ! -d pki || { echo "Existing pki/ found. Move or remove it before restore."; exit 1; }
unzip "${DOWNLOADED_PKI_ZIP}"
```

After extraction, this directory should exist:

```text
easyrsa3/pki/
```

Confirm expected PKI files are present:

```bash
ls pki
```

Expected examples:

```text
ca.crt
index.txt
issued
private
serial
```

Do not commit the restored `pki/` directory.

## Creating A Client VPN Profile

Create a separate certificate for each person:

```bash
cd "<easy_rsa_fork>/easyrsa3"
./easyrsa build-client-full <client-name>
```

Choose a strong private key passphrase when prompted. Send that passphrase
through a separate secure channel from the `.ovpn` file.

When prompted to confirm certificate details, type:

```text
yes
```

The generated `.ovpn` embeds the client private key. Passphrase protection is
required so a leaked `.ovpn` file does not immediately grant VPN access.

This creates:

```text
pki/issued/<client-name>.crt
pki/private/<client-name>.key
```

Download the base Client VPN configuration from AWS:

```text
VPC -> Client VPN endpoints -> <admin-vpn-endpoint> -> Download client configuration
```

Copy the base `.ovpn` and append the person's certificate and private key.
Prefer appending file contents directly instead of printing private keys to the
terminal:

```bash
CLIENT_NAME="<client-name>"
BASE_OVPN="<downloaded-aws-base-config>.ovpn"
CLIENT_OVPN="<client-name>.ovpn"

cp "${BASE_OVPN}" "${CLIENT_OVPN}"

cat >> "${CLIENT_OVPN}" <<'EOF'
<cert>
EOF
cat "pki/issued/${CLIENT_NAME}.crt" >> "${CLIENT_OVPN}"
cat >> "${CLIENT_OVPN}" <<'EOF'
</cert>

<key>
EOF
cat "pki/private/${CLIENT_NAME}.key" >> "${CLIENT_OVPN}"
cat >> "${CLIENT_OVPN}" <<'EOF'
</key>
EOF
```

The appended block should have this shape:

```ovpn
<cert>
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
</cert>

<key>
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
</key>
```

The resulting `.ovpn` file is a credential. Send it through a secure channel and
do not commit it to Git.

After issuing a certificate, create a fresh encrypted archive of the updated
`pki/` directory and replace the corresponding 1Password attachment:

```bash
ENV_NAME="<stage-or-prod>"
PKI_ARCHIVE="admin-${ENV_NAME}-client-vpn-pki.zip"
cd "<easy_rsa_fork>/easyrsa3"
zip -er "${PKI_ARCHIVE}" pki
```

Use the relevant environment name in the archive filename.

Upload the encrypted archive to 1Password before cleanup:

1. Open 1Password and navigate to the **Admin Client VPN PKI** item.
2. Remove the old `admin-${ENV_NAME}-client-vpn-pki.zip` attachment if present.
3. Attach the newly created `${PKI_ARCHIVE}` file.
4. Save the 1Password item.
5. Download and test-extract the attachment to confirm it is readable.

After confirming the 1Password attachment was replaced and is readable, remove
local sensitive artifacts from the working machine unless a documented secure
retention requirement exists:

Set `DOWNLOADED_PKI_ZIP` to an absolute path so cleanup still removes the
downloaded archive after changing into the `easyrsa3` directory.

```bash
DOWNLOADED_PKI_ZIP="<absolute_path_to_downloaded_pki_zip>"
cd "<easy_rsa_fork>/easyrsa3"
test -d pki || { echo "Expected easyrsa3/pki not found; aborting cleanup."; exit 1; }
rm -rf pki
rm -f "${PKI_ARCHIVE}"
rm -f "${DOWNLOADED_PKI_ZIP}"
rm -f ./*.ovpn
```

Use secure deletion tooling when available on the operator's machine. Verify the
local `pki/`, downloaded PKI zip, and generated `.ovpn` files are gone before
ending the access-change session. Clear shell history if commands or paths
included sensitive material.

## Client Onboarding

Send clients or team members these instructions:

1. Install AWS VPN Client from `https://aws.amazon.com/vpn/client-vpn-download/`.
2. Open AWS VPN Client.
3. Add a profile named `Upcyclers Admin`.
4. Import the provided `.ovpn` file.
5. Click **Connect**.
6. Open the relevant admin dashboard.

For staging:

```text
https://staging-admin.upcyclers.com
```

For production:

```text
https://admin.upcyclers.com
```

## Revoking Access

When someone leaves or no longer needs access, revoke only that person's client
certificate.

```bash
cd "<easy_rsa_fork>/easyrsa3"
./easyrsa revoke <client-name>
./easyrsa gen-crl
```

When prompted to confirm revocation, type:

```text
yes
```

This updates:

```text
pki/crl.pem
```

Import the CRL into AWS:

```text
VPC -> Client VPN endpoints -> <admin-vpn-endpoint> -> Actions -> Import client certificate CRL
```

Upload:

```text
pki/crl.pem
```

The revoked user's `.ovpn` stops working while other users remain unaffected.

After revocation, create and upload a fresh encrypted `pki/` archive to
1Password so the stored CA state remains current. After confirming the updated
1Password attachment is readable, remove local sensitive artifacts from the
working machine:

```bash
ENV_NAME="<stage-or-prod>"
PKI_ARCHIVE="admin-${ENV_NAME}-client-vpn-pki.zip"
DOWNLOADED_PKI_ZIP="<absolute_path_to_downloaded_pki_zip>"
TEMP_CRL_COPY="" # Set this to an absolute temporary CRL path if one was created.
cd "<easy_rsa_fork>/easyrsa3"
test -d pki || { echo "Expected easyrsa3/pki not found; aborting archive step."; exit 1; }
zip -er "${PKI_ARCHIVE}" pki
```

Upload the encrypted archive to 1Password before cleanup:

1. Open 1Password and navigate to the **Admin Client VPN PKI** item.
2. Remove the old `admin-${ENV_NAME}-client-vpn-pki.zip` attachment.
3. Attach the newly created `${PKI_ARCHIVE}` file.
4. Save the 1Password item.
5. Download and test-extract the attachment to confirm the updated CRL is
   present.

Only after the uploaded attachment has been verified, remove local sensitive
artifacts:

Set `DOWNLOADED_PKI_ZIP` and `TEMP_CRL_COPY` to absolute paths so cleanup still
removes those files after changing into the `easyrsa3` directory.

```bash
cd "<easy_rsa_fork>/easyrsa3"
test -d pki || { echo "Expected easyrsa3/pki not found; aborting cleanup."; exit 1; }
rm -rf pki
rm -f "${PKI_ARCHIVE}"
rm -f "${DOWNLOADED_PKI_ZIP}"
[ -z "${TEMP_CRL_COPY}" ] || rm -f "${TEMP_CRL_COPY}"
rm -f ./*.ovpn
```

Use secure deletion tooling when available and verify the local `pki/`,
downloaded PKI zip, generated `.ovpn` files, and temporary CRL copies are gone.

## DNS And Access Model

Staging admin uses a public Route 53 DNS record pointing to an internal ALB. The
hostname can resolve publicly to private `10.x.x.x` addresses, but the dashboard
is reachable only through AWS Client VPN or from inside the VPC.

We use this model because private-only DNS caused split-DNS friction on macOS
AWS VPN Client, while full-tunnel VPN would route client internet traffic
through AWS and add NAT Gateway cost and support overhead.

Expected behavior:

- VPN disconnected: DNS may resolve, but HTTPS should time out.
- VPN connected: HTTPS should reach the internal admin ALB.
- `403` with `server: awselb/2.0`: check the admin WAF allowlist and sampled
  requests.

The WAF allowlist values are managed outside this repository in AWS Secrets
Manager and AWS WAF. Do not store allowlist secret values here.
