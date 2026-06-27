# Runbook — Decommission CRM-test + rebuild UniFi on AWS Lightsail (`voz-unifi-lb`)

**Status:** ready-to-execute · **Nothing provisioned yet · $0 · all gates intact**
**Maps to vault cards:** O26 (crm-test teardown), O13 (UniFi-OS → AWS), O14 (decom `.201` Network App — *revived*, see §0), O15 (decom `.200` `unifi-os-server`), O27 (edge LB → AWS — *promoted to go per founder*).

> This runbook supersedes the original task framing in three places. Read §0 before executing anything.

---

## 0. Corrections to the original task (ground-truthed against the vault)

1. **The 4 UniFi devices are network gear, not server hosts.** The original list (`.200/.201/.218`) was the *fleet hosts*. The actual adopted devices are:

   | # | Role | Model | IP |
   |---|------|-------|-----|
   | 1 | Gateway | USG-3P (UGW3) | 192.168.1.1 |
   | 2 | Switch | USW-Lite-8-PoE | 192.168.1.6 |
   | 3 | AP | UAP-AC-Pro (U7PG2) | 192.168.1.7 |
   | 4 | AP | UAP-AC-Lite | 192.168.1.8 |

2. **The devices currently live on `.201`'s standalone UniFi Network Application**, not `.200`. They were cut over `.200 → .201` on 2026-06-18. Therefore:
   - **`.200`'s `unifi-os-server` is a stale parallel controller managing nothing** → decommissioning it is *low-risk cleanup*, **not** the irreversible device-orphaning event the task described.
   - **The real irreversible cutover is `.201 → AWS`.** Config export must come from **`.201`** (the live controller).
   - This **revives O14** (decom `.201` Network App), which the vault marked "superseded" on the wrong assumption that UniFi lived on `.200`.

3. **Edge LB → AWS is confirmed (founder go), overriding North-Star's "Traefik edge stays on `.200`."** This drags in a DNS/public-IP cutover (see §5b) — a large blast-radius step that is **out of the `voz-lightsail-provisioner` key's scope** (route53 denied) and must be a separate, gated founder action.

---

## 1. Preconditions & scope checks (run FIRST, read-only)

**AWS identity & key scope** — the `voz-lightsail-provisioner` key (acct `410244537019`) is **Lightsail-only**; `s3` / `ec2` / `route53` are **denied** (verified 2026-06-24). Before relying on it:

```bash
aws sts get-caller-identity                      # expect Account 410244537019
# Probe the exact Lightsail actions this runbook needs:
aws lightsail get-regions --query 'regions[0]' >/dev/null && echo "read OK"
```

Confirm these Lightsail actions are permitted (the O24 capture noted the scoped policy may exclude **`DeleteInstance`**):
- `CreateInstances`, `AllocateStaticIp`, `AttachStaticIp`, `OpenInstancePublicPorts`, `GetInstanceAccessDetails`
- `DeleteInstance`, `ReleaseStaticIp`, `DetachStaticIp`, `GetInstanceSnapshots`

> ⚠️ **If `DeleteInstance`/`ReleaseStaticIp` are denied**, Step 1 cannot run via this key → either widen the scoped policy (founder) or perform the teardown from the Lightsail console.

> ⚠️ **S3 is denied to this key** → the "archive config off-box to S3" option is unavailable with the provisioner key. Archive the UniFi config to the **`.201` vault** instead (§3), or use a different credential for S3.

---

## 2. STEP 1 — Tear down `voz-crm-test`  *(destructive → founder go required in-the-moment)*

**Target:** `voz-crm-test`, us-east-1a, `medium_3_0`, static IP **`54.165.37.67`**.
**Why care:** per 2026-06-25 change-log this box holds **live prod secrets** — OpenAI org-admin `sk-admin` (md5 `b4b60a5e`), real Twilio SID/token/number, Google Speech key — and can place **billed** Twilio calls.

### 2a. Verify before destroying
```bash
aws lightsail get-instance --instance-name voz-crm-test \
  --query 'instance.{name:name,az:location.availabilityZone,ip:publicIpAddress,state:state.name}'
# expect publicIpAddress == 54.165.37.67, az us-east-1a
aws lightsail get-static-ip --static-ip-name <crm-test-static-ip-name>   # confirm it maps to this instance
aws lightsail get-instance-snapshots \
  --query "instanceSnapshots[?fromInstanceName=='voz-crm-test'].name"     # MUST be empty — snapshots persist secrets!
```
**DNS check (founder-side / GoDaddy):** confirm **no `crm-test.vozqs.com` A-record exists** (deploy stopped at the DNS gate — none was ever set). Nothing to remove; just verify.

### 2b. Secret-shred (only if executing from a host with SSH to the box)
```bash
# From a fleet-side executor that can SSH the box:
ssh ubuntu@54.165.37.67 'sudo systemctl stop voz-crm || true; \
  sudo shred -u /opt/voz-crm/.env /opt/voz-crm/config/.env 2>/dev/null; \
  sudo -u postgres psql quantum_crm_prod -c "UPDATE settings SET value=NULL WHERE key IN (\
    '\''app'\'','\''openaiAdminKey'\'');" || true'
```
> From a cloud session with **no LAN/SSH**, skip 2b: `DeleteInstance` destroys the SSD, so secrets die with the disk — **provided 2a confirmed zero snapshots/AMIs**. The snapshot check is the real safeguard, not the shred.

### 2c. Destroy + release  *(irreversible)*
```bash
aws lightsail delete-instance --instance-name voz-crm-test
aws lightsail release-static-ip --static-ip-name <crm-test-static-ip-name>
```

### 2d. Confirm + report
```bash
aws lightsail get-instance --instance-name voz-crm-test 2>&1 | grep -q NotFound && echo "instance GONE"
aws lightsail get-static-ips --query "staticIps[?ipAddress=='54.165.37.67']"   # expect []
```
**Report:** instance destroyed · IP `54.165.37.67` released · 0 snapshots survived · no DNS record existed.

---

## 3. Back up the LIVE UniFi config (from `.201`, NOT `.200`)  *(read-only)*

```bash
# On .201 — export the standalone Network Application backup (the live controller of record):
#   UI: Settings → System → Backups → Download, OR the unifi backup CLI inside the container.
# Archive to the .201 vault (S3 denied to provisioner key):
cp <unifi-backup>.unf /opt/voz/vault/10-Infrastructure/backups/unifi-201-$(date +%Y%m%d).unf
( cd /opt/voz/vault && git add -A && git commit -m "backup: UniFi .201 controller config pre-AWS-cutover" )
```
> Also grab `.200`'s `unifi-os-server` config for completeness before §6 cleanup (it manages nothing, but free insurance).

---

## 4. STEP 2 — Provision `voz-unifi-lb`  *(MONEY ACTION → stop-and-report before running)*

**Spec:** `voz-unifi-lb` · us-east-1a · **`medium_3_0` ($24/mo · 2 vCPU / 4 GB / 80 GB SSD)** · `ubuntu_24_04` · static IP.
**Sizing note:** 4 GB is UniFi-OS-Server's *recommended floor*, and this box also becomes the **sole public edge** (Traefik). Fine for 4 devices + light traffic; `large_3_0` (8 GB, $44) is the headroom option if the edge grows. Founder confirmed `$24`.

### 4a. Create instance with launch script
```bash
aws lightsail create-instances \
  --instance-names voz-unifi-lb \
  --availability-zone us-east-1a \
  --blueprint-id ubuntu_24_04 \
  --bundle-id medium_3_0 \
  --user-data file://voz-unifi-lb-userdata.sh
```

`voz-unifi-lb-userdata.sh` (UniFi-OS-Server requires **Podman, NOT Docker**):
```bash
#!/usr/bin/env bash
set -euxo pipefail
apt-get update && apt-get install -y podman curl ca-certificates
# --- UniFi OS Server (native, via Podman per UI requirements) ---
# Install per Ubiquiti's self-host installer (https://help.ui.com/.../Self-Hosting-UniFi).
# Pin the UniFi OS Server release; let it own its internal ports (console 11443).
# --- Traefik v3.4 (edge LB / reverse proxy) ---
useradd -r -s /usr/sbin/nologin traefik || true
install -d /opt/load-balancer/dynamic
curl -fsSL https://github.com/traefik/traefik/releases/download/v3.4.0/traefik_v3.4.0_linux_amd64.tar.gz \
  | tar -xz -C /usr/local/bin traefik
# traefik static config: entrypoints :80/:443 (+UDP 3478/10001/10003 for UniFi STUN),
# file provider /opt/load-balancer/dynamic/*.yml — mirrors the .200 edge layout.
```

### 4b. Static IP + firewall
```bash
aws lightsail allocate-static-ip --static-ip-name voz-unifi-lb-ip
aws lightsail attach-static-ip --static-ip-name voz-unifi-lb-ip --instance-name voz-unifi-lb
# Firewall: Traefik edge + UniFi-OS-Server port set
for P in 80 443 11443 8080 8444 8880 8881 8882 9543 6789 5005 5514 10003; do
  aws lightsail open-instance-public-ports --instance-name voz-unifi-lb \
    --port-info fromPort=$P,toPort=$P,protocol=TCP; done
for U in 3478 10003; do
  aws lightsail open-instance-public-ports --instance-name voz-unifi-lb \
    --port-info fromPort=$U,toPort=$U,protocol=UDP; done
# Restrict 22 (SSH) to the founder/.201 source CIDR — do NOT leave 0.0.0.0/0.
```

> ⚠️ **Port-binding conflict to resolve on-box:** Traefik (edge) wants **:443**; UniFi-OS-Server also defaults to **:443/:11443**. Let **Traefik own 80/443** and reverse-proxy a UniFi subdomain (e.g. `unifi.vozqs.com`) to UniFi-OS-Server's console on **11443** internally. Confirm UniFi-OS-Server is bound to 11443/8443 (not 443) before wiring Traefik.

**Report after §4:** instance up · static IP attached (note the new public IP) · UniFi-OS-Server console reachable on 11443 · Traefik answering on 80/443.

---

## 5a. STEP 3 — UniFi device cutover  `.201 → AWS`  *(device cutover → explicit founder go in-the-moment; REQUIRES LAN/SSH)*

> **Cannot run from a cloud session** — needs SSH to `.201` and L2 reach to the devices (192.168.1.x). Fleet-side executor only.

1. **Restore** the `.201` backup (§3) into UniFi-OS-Server on `voz-unifi-lb` (import via console). This carries device config/settings forward.
2. **Re-inform each device** to the new controller. For each of the 4 (USG `.1`, USW-Lite-8 `.6`, AC-Pro `.7`, AC-Lite `.8`):
   ```bash
   # SSH to the device (UniFi device SSH creds), then:
   set-inform http://<voz-unifi-lb-private-or-reachable-ip>:8080/inform
   ```
   > **Reachability caveat:** devices are on the LAN behind NAT; the AWS controller is public. `set-inform` must point at an address the devices can reach. Either (a) site-to-site VPN / WireGuard from LAN → `voz-unifi-lb`, or (b) port-forward `:8080` inform + adoption through the gateway. **Adopting on-prem L2 devices to a cloud controller is non-trivial** — validate the inform path before cutover. This is the single biggest execution risk in the whole runbook.
3. **Adopt** each device in the new controller UI; wait for `Connected`.
4. **Verify:** all 4 devices `Connected` + provisioned · clients online · no WAN/Wi-Fi drop on the gateway.

---

## 5b. STEP 3 (edge) — Move live traffic to the `voz-unifi-lb` Traefik edge  *(LARGE blast radius → separate founder gate)*

> This is the O27 piece. It is **bigger than a box deploy** and **outside the provisioner key's scope** (route53/DNS denied).

1. **Mirror the `.200` Traefik dynamic config** (`/opt/load-balancer/dynamic/*.yml`) onto `voz-unifi-lb`, repointing backends (vozqs.com, crm, mail, bot, ui, …) at their on-prem hosts over the internet.
2. **Validate routing WITHOUT cutting live traffic** — host-header test against the new IP first:
   ```bash
   curl -sk -H 'Host: vozqs.com' https://<voz-unifi-lb-ip>/ -o /dev/null -w '%{http_code}\n'   # expect 200
   ```
   This satisfies the task's "test one downstream" safely, pre-DNS-flip.
3. **Telephony stays on-prem** (North-Star): SIP/RTP + WebRTC softphone remain on `.200`. Ensure the new edge does **not** intercept telephony entrypoints, or that they bypass it.
4. **DNS cutover (founder, GoDaddy/Route53):** repoint A-records `50.89.183.227 → <voz-unifi-lb-ip>`. **Founder action — key can't touch DNS.** Low TTL first; flip; watch; rollback = revert A-record.

---

## 6. STEP 2-equiv cleanup — decommission the old controllers  *(after §5a proven · founder go)*

- **`.201` Network Application** (O14, revived): once devices are healthy on AWS, stop + disable `unifi-network-application` + `unifi-db` containers. Keep the `.201` backup (§3).
- **`.200` `unifi-os-server`** (O15): stale parallel controller — stop + disable. Manages nothing post-cutover; low risk. Also remove the `.200` Traefik `ui/unifi → unifi-os-server:443` route once §5b is live.

---

## 7. Gate summary

| Step | Action | Gate |
|------|--------|------|
| §2 | Delete crm-test + release IP | **founder go in-the-moment** (irreversible; prod secrets) |
| §4 | Provision `voz-unifi-lb` | **stop-and-report** (money $24/mo) |
| §5a | Device cutover `.201 → AWS` | **founder go in-the-moment** (live network devices) |
| §5b | DNS/edge live-traffic flip | **separate founder gate** (large blast radius; DNS = founder) |
| §6 | Decom `.201` + `.200` controllers | **founder go**, after §5a proven |

**Secrets:** referenced by name only throughout. **Rollback:** none for §2 (founder-accepted, no parallel-run); §5b rollback = revert DNS A-record.

---

## 8. Known-unknowns to resolve before/at execution

1. `voz-lightsail-provisioner` policy: are `DeleteInstance` / `ReleaseStaticIp` in scope? (§1)
2. crm-test static-IP **resource name** (vs the `54.165.37.67` value) — needed for release.
3. On-prem → cloud **inform/adoption path** for the 4 devices (VPN vs port-forward) — §5a, highest risk.
4. UniFi-OS-Server vs Traefik **:443 binding** resolution — §4b.
5. `voz-unifi-lb` **new public IP** → drives the §5b DNS change.
6. Is the 4th device confirmed as **UAP-AC-Lite**? (inventory: UGW3 + USW-Lite-8 + UAP-AC-Pro + UAP-AC-Lite = 4 ✓)
