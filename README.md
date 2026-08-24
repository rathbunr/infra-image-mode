# RHEL 10 Image Mode (bootc) — CIS Level 1 + FIPS → Satellite Pipeline

Configuration-as-Code pipeline that:

1. Builds an immutable **RHEL 10 bootc** image (CIS Level 1 Server, FIPS 140-3, PIV/smart card)
2. Pushes it to a private registry
3. Configures **Red Hat Satellite** with an Image Mode Kickstart template and Host Group

Designed for **CMMC Level 2** / STIG-aligned environments.

---

## Artifacts

| Artifact | Path |
|----------|------|
| Containerfile + FIPS kargs | `container/Containerfile`, `container/01-fips.toml` |
| Build & push | `playbooks/01-build-push-image.yml` (`containers.podman`) |
| Kickstart (ostreecontainer) | `templates/kickstart_rhel10_bootc_cis.erb` |
| Satellite config | `playbooks/02-configure-satellite.yml` (`redhat.satellite`) |
| Orchestrator | `playbooks/site.yml` |

---

## 1. Immutable OS container

- **Base:** `registry.redhat.io/rhel10/rhel-bootc:latest`
- **CIS Level 1:** `oscap-im --profile xccdf_org.ssgproject.content_profile_cis` on RHEL 10 SSG datastream
- **FIPS:** `update-crypto-policies --set FIPS` + `/usr/lib/bootc/kargs.d/01-fips.toml` (`kargs = ["fips=1"]`)
- **Smart card / YubiKey PIV:** `opensc`, `pcsc-lite`, `pcsc-lite-ccid`; `pcscd` enabled; `authselect` with smartcard

No traditional package set at install time — everything is in the image.

---

## 2. How Satellite passes registry credentials securely

Image Mode Kickstart uses `ostreecontainer --url <image>` instead of `%packages`.  
If the registry requires authentication:

1. Store a **hidden Host Group / Host parameter** `registry_auth_b64` whose value is the  
   `auth` field from [containers-auth.json(5)](https://github.com/containers/image/blob/main/docs/containers-auth.json.5.md)  
   (typically `base64(username:token)`).
2. The provisioning template’s **`%pre`** block writes:

   ```text
   /etc/ostree/auth.json
   ```

   with that value, scoped to your registry host. Anaconda/bootc use this file **only in the installer environment** to pull the image.
3. **`%post`** tightens permissions so the deployed system can still run `bootc upgrade` against the same private registry.

Credentials never appear in the PXE kernel command line. Prefer short-lived registry tokens and rotate the host parameter via CaC/vault.

Kernel option note: do **not** use Kickstart `url=`; for bootc/ostreecontainer set `inst.stage2=` as needed for the boot ISO.

---

## 3. Run the pipeline

```bash
# Collections
ansible-galaxy collection install -r collections/requirements.yml

# Secrets
cp files/vault-example.yml files/vault.yml
ansible-vault encrypt files/vault.yml

# Full pipeline
ansible-playbook -i inventory/hosts.yml playbooks/site.yml \
  -e @files/vault.yml --ask-vault-pass
```

Then in Satellite:

1. Set Host Group parameter `registry_auth_b64` (hidden) if the registry is authenticated.
2. Create a host in Host Group **RHEL 10 bootc CIS L1 FIPS** and provision (PXE or boot ISO + `inst.ks=`).

---

## 4. Compliance notes

| Control | Implementation |
|---------|----------------|
| CIS Level 1 Server | `oscap-im` at image build |
| FIPS 140-3 | Crypto policy + `fips=1` karg baked into image |
| Immutable OS | bootc / ostree container; no `%packages` |
| PIV / smart card | packages + `pcscd` + authselect |
| Secrets | Vault + hidden Satellite parameters; `no_log` on sensitive tasks |
| SELinux | Enforcing in image |

Post-provision: re-run OpenSCAP against the live system for audit evidence; some build-time rules differ from runtime.

---

## References

- [Image mode for RHEL](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/)
- [FIPS while building a bootc image](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/enabling-the-fips-mode-while-building-a-bootc-image)
- [Provisioning image mode with Satellite](https://developers.redhat.com/articles/2026/08/06/image-mode-for-rhel-with-anaconda-and-satellite)
- [Security hardening of bootable images (`oscap-im`)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/security-hardening-and-compliance-of-bootable-images)
