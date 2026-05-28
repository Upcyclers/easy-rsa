# Upcyclers Admin Client VPN PKI Runbook

This EasyRSA fork is used as the tool source for managing Upcyclers admin AWS
Client VPN certificates. It is not the source of truth for generated VPN
credentials.

Do not commit generated PKI state, `.ovpn` files, private keys, certificate
revocation lists, or encrypted PKI archives to Git.

## Current Access Model

Production admin is available at:

```text
https://admin.upcyclers.com
```

The hostname is private-only DNS in Route 53 and resolves through the production
VPC DNS resolver:

```text
10.0.0.2
```

Administrators must connect through AWS Client VPN before opening the admin
dashboard. The public website certificate `*.upcyclers.com` is for the admin ALB
HTTPS listener only; it is not valid for AWS Client VPN mutual authentication.

## Production Resources

- Client VPN endpoint: `upcyclers-prod-admin-vpn`
- Client VPN DNS resolver: `10.0.0.2`
- Client VPN client CIDR: `172.20.8.0/22`
- Production VPC CIDR: `10.0.0.0/16`
- Private hosted zone: `admin.upcyclers.com`
- Admin WAF Web ACL: `admin-prod`
- WAF allowlist secret: `admin/prod/ip-allowlist`
- Client VPN log group: `/aws/clientvpn/upcyclers-prod-admin-vpn`

## Responsibility

The admin technical owner, CTO, or delegated infrastructure developer is
responsible for admin VPN certificate operations.

That owner is responsible for:

- Maintaining the encrypted EasyRSA `pki/` archive in 1Password.
- Generating one client certificate per administrator, unless a documented
  shared owners profile is intentionally used.
- Creating personalized `.ovpn` profiles.
- Sending `.ovpn` files only through approved secure channels.
- Revoking certificates when access should be removed.
- Importing updated CRLs into AWS Client VPN.

Administrators should not run EasyRSA themselves. They should only install AWS
VPN Client, import the provided `.ovpn` profile, connect, and open the admin
dashboard.

## Storage Model

This repository stores only EasyRSA source code and Upcyclers operating
instructions.

Sensitive state lives in 1Password:

```text
1Password item: Admin Client VPN PKI
```

The 1Password item should store the encrypted production PKI archive:

```text
admin-prod-client-vpn-pki.zip
```

The generated `pki/` directory is the actual certificate authority state. It
contains the CA key, issued certificates, client private keys, serial/index
database, and CRL state. Losing it makes per-user issuance and revocation
difficult; leaking it compromises VPN access.

The repository `.gitignore` ignores:

```text
easyrsa3/pki
```

Do not bypass that ignore rule.

## Restore PKI State

To issue or revoke VPN certificates, restore the production PKI archive from
1Password:

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

## Certificate Model

There is one production admin VPN CA/PKI. Each administrator should normally get
their own client certificate/key pair.

Correct mapping:

```text
VPN endpoint server cert = pki/issued/server.crt
VPN endpoint server key  = pki/private/server.key
Client cert              = pki/issued/<client-name>.crt
Client key               = pki/private/<client-name>.key
CA cert                  = pki/ca.crt
```

Never put these into a user's `.ovpn`:

```text
pki/private/ca.key
pki/private/server.key
pki/issued/server.crt
```

## Create A Client VPN Profile

Create a separate certificate for each administrator:

```bash
cd "<easy_rsa_fork>/easyrsa3"
./easyrsa build-client-full <client-name>
```

Use a clear client name, for example:

```text
ghian-upcyclers
alex-prod-admin
```

Choose a strong private key passphrase when prompted and store it in 1Password.
If AWS VPN Client has trouble with encrypted client keys, regenerate that user's
certificate with `nopass` after documenting the tradeoff:

```bash
./easyrsa build-client-full <client-name> nopass
```

This creates:

```text
pki/issued/<client-name>.crt
pki/private/<client-name>.key
```

Download a fresh base Client VPN configuration from AWS:

```text
VPC -> Client VPN endpoints -> upcyclers-prod-admin-vpn -> Download client configuration
```

Copy the base `.ovpn` and append the person's certificate and private key:

```bash
CLIENT_NAME="<client-name>"
BASE_OVPN="<downloaded-prod-base-config>.ovpn"
CLIENT_OVPN="upcyclers-prod-admin-${CLIENT_NAME}.ovpn"

cp "${BASE_OVPN}" "${CLIENT_OVPN}"
```

Add these lines outside any certificate blocks:

```ovpn
mssfix 1200
tun-mtu 1400
```

They are required for the current production VPN path. Without them, clients may
connect to the admin ALB TCP port but hang during TLS handshake.

Append only the PEM certificate and key blocks:

```bash
cat >> "${CLIENT_OVPN}" <<EOF

<cert>
$(sed -n '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/p' "pki/issued/${CLIENT_NAME}.crt")
</cert>

<key>
$(cat "pki/private/${CLIENT_NAME}.key")
</key>
EOF
```

The resulting `.ovpn` file is a credential. Send it through a secure channel and
do not commit it to Git.

## Archive Updated PKI

After issuing a certificate, create a fresh encrypted archive of the updated
`pki/` directory and replace the 1Password attachment:

```bash
ENV_NAME="prod"
PKI_ARCHIVE="admin-${ENV_NAME}-client-vpn-pki.zip"
cd "<easy_rsa_fork>/easyrsa3"
zip -er "${PKI_ARCHIVE}" pki
```

Upload the encrypted archive to 1Password before cleanup:

1. Open 1Password and navigate to the **Admin Client VPN PKI** item.
2. Remove the old `${PKI_ARCHIVE}` attachment if present.
3. Attach the newly created `${PKI_ARCHIVE}` file.
4. Save the 1Password item.
5. Download and test-extract the attachment to confirm it is readable.

After confirming the 1Password attachment was replaced and is readable, remove
local sensitive artifacts from the working machine unless a documented secure
retention requirement exists.

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
ending the access-change session.

## Administrator Setup

Send administrators these instructions:

1. Install AWS VPN Client from `https://aws.amazon.com/vpn/client-vpn-download/`.
2. Open AWS VPN Client.
3. Add a profile named `Upcyclers Prod Admin`.
4. Import the provided `.ovpn` file.
5. Click **Connect**.
6. Open `https://admin.upcyclers.com`.

For macOS, add a resolver if `admin.upcyclers.com` does not resolve through the
VPN:

```bash
sudo mkdir -p /etc/resolver
sudo sh -c 'printf "nameserver 10.0.0.2\n" > /etc/resolver/admin.upcyclers.com'
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

This resolver only affects `admin.upcyclers.com`.

Verification:

```bash
dscacheutil -q host -a name admin.upcyclers.com
curl -v --http1.1 --connect-timeout 10 --max-time 30 https://admin.upcyclers.com/api/health
```

Expected:

```text
admin.upcyclers.com resolves to 10.0.x.x
HTTP response is 200
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
VPC -> Client VPN endpoints -> upcyclers-prod-admin-vpn -> Actions -> Import client certificate CRL
```

Upload:

```text
pki/crl.pem
```

The revoked user's `.ovpn` stops working while other users remain unaffected.

After revocation, create and upload a fresh encrypted `pki/` archive to
1Password so the stored CA state remains current.

If a shared owners profile is used and any owner leaves or loses the profile,
revoke the shared certificate and issue a new owners profile to the remaining
owners.

## Troubleshooting

- VPN TLS handshake fails before connection: Client VPN endpoint is using the
  wrong ACM cert, the `.ovpn` client cert/key do not match, or the client cert
  was not signed by the endpoint's trusted CA.
- `Could not resolve host`: macOS resolver is missing or VPN DNS is not active.
- TCP connects to the ALB but TLS hangs: confirm `mssfix 1200` and
  `tun-mtu 1400` are present in the imported `.ovpn`.
- `403` with `server: awselb/2.0`: check WAF sampled requests for `admin-prod`.
- Health check works from an EKS debug pod but not from VPN: check Client VPN
  routes, authorization rules, security groups, and profile MTU settings.
