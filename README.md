# Mist Entra Access Assurance (simple)

This is a simplified edition of `mist-entra-access-assurance` (frozen at v1.2.0). One command connects **Juniper Mist Access Assurance** to **Microsoft Entra ID** and gives users two ways to onboard: a PSK self-service portal and a certificate (Marvis Client) portal. A second command puts the Mist org and the Entra tenant back exactly as they were.

The site setup, floorplan, Entra group/user creation and `--pause` from v1.2.0 have been removed. The script uses an existing site, and optionally an existing Entra group, without changing either.

## What it builds

| Step | What happens |
|---|---|
| 1. Connect | Signs in to Entra ID and Mist, checks the site (and the group, if set), and saves a snapshot |
| 2. Entra app | **IdP Connector** app registration with Graph `User.Read` (delegated) and `User.Read.All`, `GroupMember.Read.All`, `Device.Read.All` (application), plus admin consent and public client flows (ROPC / EAP-TTLS) |
| 3. Mist IdP | OAuth/Azure identity provider with a fresh client secret (never shown), plus realm mapping and the default IdP |
| 4. Internal CA | Turns on the Mist onboard CA for BYOD certificates |
| 5. SSIDs | A Multi-PSK SSID and an 802.1X SSID on Mist NAC, each in its own WLAN template. Both are applied to `[site] name` |
| 6. Auth policy | Rule *EAP-TLS → allow*, or *EAP-TLS + group → allow* when `user_group` is set |
| 7. Entra SAML | Policies that sign the whole SAML response and send the UPN as an email-format NameID. Mist requires both |
| 8. PSK portal | BYOD self-service portal with Entra SSO (no email collected or sent), plus its SAML app |
| 9. Cert portal | Marvis Client certificate onboarding portal with Entra SSO, plus its SAML app |

Each build writes a **Word report** with both portal QR codes and every action taken. Every run also writes an **audit log**, a copy of the screen output with a time on each line. Both file names carry the run's timestamp.

## Group (optional)

`[entra] user_group` names an **existing** Entra security group. The script never creates or changes it.

- **Set:** only the group's members can sign in to the portals (via app assignment, which needs Entra ID P1; without P1 the script warns and opens the portals to everyone), and the NAC rule also requires membership.
- **Blank:** any user in the tenant can onboard.

## Setup

```bash
pip3 install -r requirements.txt
cp mist_entra_aa_simple.ini.example mist_entra_aa_simple.ini
```

In the ini, set the Mist `cloud` + `token` (or `credentials` + `label` from a credential store), plus the Entra `tenant`, the site `name` and the SSID names. Mist object names must be **32 characters or fewer**. The script checks this before it starts.

## Run

```bash
python3 mist_entra_aa_simple.py --dry-run
```

```bash
python3 mist_entra_aa_simple.py
```

```bash
python3 mist_entra_aa_simple.py --teardown
```

- `--dry-run` only reads, and shows every step it would take.
- `--teardown` restores everything from the snapshot. Add `--yes` to skip the prompt.
- `--rotate-secret` sends a new Entra client secret to the existing Mist identity provider.

Entra sign-in uses a device code; sign in as a Global Administrator. The token is cached in `.msal_token_cache.json` (chmod 600). Re-running the build is safe: anything that already exists is skipped.

## Teardown

The snapshot, `<ini>.state.json`, is written before the first change. Teardown then:

- Deletes every Mist object the build created, plus any PSKs issued on the PSK SSID.
- Restores the org's Access Assurance IdP settings.
- Turns the internal CA back off if it was off before, which removes every certificate it issued. If the CA was already on, it keeps it and restores its providers.
- Deletes the Entra app registrations, SAML apps and policies, and purges them from the Entra recycle bin.

Objects the build didn't create are never touched.

## Tips

- **On an org that has never run Access Assurance, allow about 10 minutes after the first 802.1X join attempt.** That attempt makes Mist build the org's cloud RADIUS service; until it's ready, joins fail with 514/515. Step 1 tells you if the org has no Access Assurance traffic yet.
- After a rebuild, open the portal links in a **new private tab**. The portals cache a sign-in that belongs to the previous (deleted) portal.
- EAP-TTLS (username/password) uses ROPC, which skips MFA.
- The site needs an AP for the SSIDs to broadcast.
