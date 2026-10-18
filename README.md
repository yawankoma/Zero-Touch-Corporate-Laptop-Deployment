# Zero-Touch Corporate Laptop Deployment (Windows Autopilot + Intune)

**One line summary:** A cloud-only setup where a new company laptop is unboxed, signed into, and ends up managed, protected and automatically checked for compliance, with no IT hands-on time per device.

**Scenario:** A fictional company hires staff and needs laptops that are locked down from the first sign in. Everything runs on Microsoft Entra ID and Intune, with no on-premises Active Directory.

**My role:** Endpoint administrator (design, build, test, document).
**Lab:** Windows 11 Pro virtual machine (VMware), Microsoft 365 tenant with Intune.

---

## The business problem

| Risk | What it costs a bank |
|---|---|
| Laptops set up by hand | Slow onboarding, inconsistent security, high IT workload |
| Unmanaged or out-of-date devices reaching company data | Data breaches, fraud losses, failed audits |
| No proof that devices meet policy | Compliance findings and regulatory fines |

## What I built

| Area | What I configured | Why it matters |
|---|---|---|
| Zero-touch enrollment | Windows Autopilot (user driven, Entra join), hardware hash registered with group tag `CORP`, dynamic device group, deployment profile | New laptops enroll themselves; scales from 1 to 500 devices |
| Sign-in security | Windows Hello for Business (PIN bound to the device), phone sign-in disabled | Credentials tied to the device, fewer attack paths |
| Endpoint protection | Microsoft Defender Antivirus policy, BitLocker disk encryption policy with recovery keys stored in Entra ID | Malware protection and encryption on lost or stolen devices |
| App deployment | Microsoft 365 Apps (Word, Excel, PowerPoint, Outlook, Teams) pushed as a required app | Staff are productive at first sign in |
| Compliance | Device compliance policy: BitLocker encryption, sign in credential, firewall, antivirus, antispyware, Defender real time protection, up to date signatures, Windows 11 minimum | Defines what a "trusted" device is |
| Access control | Conditional Access policy: require a compliant device for cloud apps (tested in Report-only mode) | Only trusted devices reach company data |
| Patching | Two update rings: Pilot (updates immediately) and Broad (7 day quality, 30 day feature deferral), 30 day uninstall window, enforced deadlines | A bad update is caught on the pilot group before it reaches everyone |

## Key decisions and trade-offs

- **Group tag + dynamic group instead of manual assignment.** Devices are labelled `CORP` when registered, so policies follow the label. My first attempt used a hardware and OS version rule, which failed because OS version does not exist on the device record before enrollment.
- **Report-only before enforcing.** The Conditional Access policy was tested in Report only mode and my admin account was excluded, so a mistake could not lock out the tenant. A production setup would also keep a separate break glass account.
- **Immediate non-compliance flag.** Failing devices are marked non compliant at once for fast testing. Production would add a grace period (around 1 day) and a notification email, to avoid disrupting users unnecessarily.
- **Windows Hello TPM setting.** Set to Preferred because the lab VM has no virtual TPM. Production would use Required.
- **BitLocker kept as a compliance requirement.** I deliberately did not weaken the policy to make the lab VM pass. The VM has no TPM, so it is flagged non-compliant, which is the policy doing its job (see the case study below).
- **Conflicting policies avoided.** Defender created a default antivirus policy for all devices, so custom policies are scoped to the corporate group only.
- **Pilot group excluded from the Broad ring.** A device only ever receives one update ring, which prevents conflicting update policies.

## Case study: why the lab device is non-compliant

1. The compliance policy requires BitLocker encryption.
2. The BitLocker policy encrypts silently and only with a TPM (startup without a TPM is not allowed, so users never see a password or USB key prompt).
3. The lab VM has no virtual TPM, so the Intune encryption report shows it as **Not ready** and **Not encrypted**.
4. Because the device cannot meet the encryption requirement, Intune marks it **non-compliant**, and the Conditional Access policy (Report-only) shows it **would be blocked** from cloud apps.

**What this shows:** the controls work as designed. A device that enrolled correctly but is not encrypted does not get trusted access.

**Production fix:** a corporate laptop with TPM 2.0 encrypts silently after sign in, stores its recovery key in Entra ID, and becomes compliant with no user action.

## Proof

> Replace each line below with your screenshot. Blur the tenant name, user email domain and device serial number first.

1. **Autopilot registration:** device listed with group tag `CORP` and profile status Assigned.
   `![Autopilot device](screenshots/01-autopilot-device.png)`
2. **Dynamic group rule:** group rule on the `[OrderID]:CORP` tag, VM listed as a member.
   `![Dynamic group](screenshots/02-dynamic-group.png)`
3. **Out-of-box experience:** the organization-branded sign-in screen during setup.
   `![OOBE](screenshots/03-oobe.png)`
4. **Enrolled device:** device in Intune with the user, "Corporate" ownership and Managed by Intune.
   `![Enrolled device](screenshots/04-enrolled-device.png)`
5. **Non-compliant state:** device marked non-compliant in Intune, with the per-setting view showing the encryption requirement as the failing item.
   `![Non-compliant](screenshots/05-non-compliant.png)`
6. **Conditional Access result:** sign-in log Report-only tab showing the policy **would fail** for this device.
   `![CA report-only](screenshots/06-ca-report-only.png)`
7. **Encryption report:** shows the VM as not ready for BitLocker (TPM unknown), which explains the non-compliance.
   `![Encryption report](screenshots/07-encryption-report.png)`
8. **Update rings:** the Pilot ring showing the VM in device status, and the Broad ring where it does not appear.
   `![Update rings](screenshots/08-update-rings.png)`

## What I found and how I fixed it

| Problem | Cause | Fix |
|---|---|---|
| VM never joined the Autopilot group | Dynamic rule relied on OS version, which only exists after enrollment | Switched to the group tag rule |
| Policy applied to the wrong group | Profile assignment pointed at an old group as well as the right one | Removed the old group from the assignment |
| Device showed non-compliant | Compliance policy requires BitLocker, and the VM has no TPM | Confirmed the failing setting, kept the requirement on purpose, and documented it as a case study |
| BitLocker did not encrypt | No TPM on the VM | Documented as expected; production laptops encrypt silently |

## Limitations (honest)

- BitLocker encryption is configured but **not demonstrated** on the lab VM because it has no TPM, so the device stays non-compliant by design.
- Conditional Access was validated in Report only mode; it is the testing stage before enforcement.
- Single device and single test user, so no scale metrics are claimed.



## Next steps

- Add a Multifactor authentication Conditional Access policy as the identity layer next to device trust.
- Defender for Endpoint onboarding and a device risk score requirement.
- Device naming template for new enrollments (for example `CORP-` plus serial).
- Add a TPM-enabled test device so BitLocker encryption can be demonstrated and the device reaches compliant.
