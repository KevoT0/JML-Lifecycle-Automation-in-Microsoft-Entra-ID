# JML Lifecycle Automation in Microsoft Entra ID

**Platform:** Microsoft Entra ID · PowerShell · Microsoft Graph
**Domain:** Identity & Access Management · Joiner–Mover–Leaver lifecycle
**Automation surface:** Microsoft Graph PowerShell SDK (app-only authentication)

---

## The problem — a real-world attack, not a hypothetical
 
In **April 2022, Block Inc.** (parent of **Cash App**) disclosed that a **former employee had downloaded internal reports containing the personal data of roughly 8.2 million current and former US customers** — brokerage account numbers, portfolio values, and holdings. The employee had legitimate access to those reports while employed. The failure: **that access was never fully revoked when they left**, and *months* after departure they could still pull the data. [1][2]
 
This is the most preventable class of identity failure — the **orphaned account**, access that outlives the person's need for it. It shows up two ways: the **leaver** who keeps access after departure (Cash App), and the quieter **mover** who changes teams over the years and accumulates access to every team they ever passed through, none of it ever removed. Both leave standing access an attacker — or a departing insider — can walk straight through. Manual offboarding is where this breaks: at scale, humans forget, and every forgotten account is a breach waiting to happen.
 
## What this project is — and the skills it proves
 
This project builds the automation that closes that gap: an **attribute-driven Joiner–Mover–Leaver (JML) lifecycle engine** in Microsoft Entra ID, using PowerShell and Microsoft Graph. Access becomes a function of a user's attributes — granted automatically on joining, re-aligned on moving, and **stripped automatically, on the day, when they leave**. It demonstrates the identity-automation skills to make access provably match a person's *current* role at all times, unattended, at scale.
 
| Real-world failure | Control this project builds |
|---|---|
| Ex-employee retains access after departure (Cash App) | Date-gated **Leaver** automation — revoke sessions, strip all groups, disable the account on the leave date |
| Access piles up across role changes (privilege creep) | **Mover** reconciliation — add the new, strip the stale, automatically |
| New starters provisioned inconsistently by hand | **Joiner** automation — access derived from the department attribute |
| Offboarding depends on someone remembering | App-only automation runs unattended on a schedule — nothing to forget |
 
This is the core logic that IGA platforms such as SailPoint and Okta Lifecycle productise — implemented directly against Graph to demonstrate the underlying mechanics.

---

## Architecture & security model

The automation authenticates as an application, not a user, so it can run unattended on a schedule. This required a dedicated app registration.

![App registration for the automation identity](https://github.com/KevoT0/JML-Lifecycle-Automation-in-Microsoft-Entra-ID/blob/main/01-app-registration-redacted.png)

Permissions were granted following **least privilege** — only the Graph scopes each task needs, added incrementally as functionality was built, each with admin consent:

![Least-privilege API permissions](https://github.com/KevoT0/JML-Lifecycle-Automation-in-Microsoft-Entra-ID/blob/main/2.png)

| Scope | Purpose |
|---|---|
| `User.Read.All`, `Group.Read.All` | read users and groups |
| `GroupMember.ReadWrite.All` | manage group membership |
| `Group.ReadWrite.All` | create department groups (setup only) |
| `User.ReadWrite.All` | create users / disable leavers |
| `User-LifeCycleInfo.ReadWrite.All` | set leave dates |

Authentication uses a client secret. The secret is stored in a local `config.json` and **excluded from source control** via `.gitignore` — it is never hardcoded in a script or committed.

![Client secret used for app-only authentication](https://github.com/KevoT0/JML-Lifecycle-Automation-in-Microsoft-Entra-ID/blob/main/3.png)

---

## The environment

A fictional company was stood up entirely by script — three department security groups (Finance, Marketing, Engineering) and ten users, each stamped with a `department` attribute. The setup script is idempotent: it checks for existing objects before creating, so it can be re-run safely.

![Lab setup and first provisioning run](https://github.com/KevoT0/JML-Lifecycle-Automation-in-Microsoft-Entra-ID/blob/main/4.png)

---

## Joiner

A new hire is created with a `department` attribute set — the only input the automation needs.

![New joiner created with a department attribute](https://github.com/KevoT0/JML-Lifecycle-Automation-in-Microsoft-Entra-ID/blob/main/5.png)

She appears in the directory with her department populated, ready to be provisioned on the next run.

![New joiner in the directory roster](https://github.com/KevoT0/JML-Lifecycle-Automation-in-Microsoft-Entra-ID/blob/main/6.png)

The provisioning engine is a single mapping — the department-to-group policy — applied to every user:

```powershell
$deptToGroup = @{
    "Finance"     = "Finance-Team"
    "Marketing"   = "Marketing-Team"
    "Engineering" = "Engineering-Team"
}
```

The engine reads each user's department, adds them to the matching group, and removes them from any other department group. Adding a new department is a single line — no change to the engine itself.

---

## Mover

The trigger is a single manual step: an analyst changes the user's `department` (here, Amara Okafor moves from Finance to Engineering).

![Department changed as the mover trigger](https://github.com/KevoT0/JML-Lifecycle-Automation-in-Microsoft-Entra-ID/blob/main/7.png)

On the next run the script grants the new group **and strips the stale access automatically** — the line that kills privilege creep:

```
REMOVED Amara Okafor from Finance-Team (stale access)
Placed  Amara Okafor (dept: Engineering) -> Engineering-Team
```

![Mover run stripping stale access and granting the new group](https://github.com/KevoT0/JML-Lifecycle-Automation-in-Microsoft-Entra-ID/blob/main/8.png)

Every other user is simply re-confirmed in place. The engine is idempotent — it only acts on drift — so it is safe to run on a schedule.

---

## Leaver

Leavers are flagged with a leave date (`EmployeeLeaveDateTime`), simulating what an HR feed would populate in production. The tagging is deliberately a human decision — a person confirms who is leaving and when — while the offboarding itself is fully automated.

![Tagging leavers with a leave date](https://github.com/KevoT0/JML-Lifecycle-Automation-in-Microsoft-Entra-ID/blob/main/9.png)

The `Leaver.ps1` engine is date-gated. It contains no names and no dates; it reads each user's leave date and compares it to now:

```powershell
if ($leaveDate -gt $now) { continue }   # not yet due — skip, access preserved
# else: revoke sessions -> strip all groups -> disable account
```

On a run, users whose leave date has passed are fully offboarded — sessions revoked, all groups stripped, account disabled — while users with a future date are correctly skipped:

![Leaver run — date-gated offboarding](https://github.com/KevoT0/Entra-JML-Automation/blob/main/10.png)

A contractor tagged to leave on the 30th keeps full access until that date, then is offboarded automatically on the next scheduled run. The offboarding is deliberately three steps — revoke sessions (kill active tokens immediately), strip all group memberships, then disable the account. Accounts are **disabled, not deleted**, preserving them for audit and legal retention; deletion is handled separately after a retention period.

---

## Key design decisions

- **Human sets the attribute, the machine enforces the access.** Every stage follows the same pattern: a person sets one attribute (`department`, or a leave date); the automation derives and applies the correct access. This keeps the human decision where it belongs and removes the error-prone, repetitive execution.
- **Idempotent by design.** The engines can run repeatedly and only change what has drifted, making them safe to schedule (e.g. Joiner/Mover every 30 minutes, Leaver daily).
- **Least privilege, incrementally.** Each Graph scope was added only when a task required it — demonstrated live when unscoped operations correctly returned `403 Forbidden`.
- **Disable, don't delete.** Leaver accounts are disabled and stripped, not deleted, preserving forensic and compliance value.
- **Secrets never committed.** Credentials live in a git-ignored `config.json`; the repo ships a `config.json.example` template instead.

This is the core logic that IGA platforms such as SailPoint and Okta Lifecycle productise — implemented directly against Graph to demonstrate the underlying mechanics.

---

## Future improvements

- **Scheduling** — Windows Task Scheduler for the lab; an Azure Automation runbook for a production, always-on deployment.
- **HR-driven input** — in production, `department` and `EmployeeLeaveDateTime` would be populated from an HR system (e.g. Workday) via inbound provisioning rather than set manually.
- **Reconciliation reporting** — a scheduled audit flagging any user whose group membership does not match their department, or any past-leave-date account still enabled, with a monthly JML summary export.
- **Access reviews** — periodic manager attestation that access is still required (Entra Access Reviews).

---

## Skills demonstrated

· Identity lifecycle automation (Joiner–Mover–Leaver)
· PowerShell + Microsoft Graph SDK
· App-only authentication (client credentials flow)
· Least-privilege permission design
· Attribute-driven provisioning
· Privilege-creep remediation
· Secure secret handling
· Idempotent, schedulable automation


## References
 
1. TechCrunch — [Block confirms Cash App breach after former employee accessed US customer data](https://techcrunch.com/2022/04/05/block-cash-app-data-breach/) (April 2022).
2. Security.org — [Cash App Data Breach: What Happened and What to Do](https://www.security.org/identity-theft/breach/cash-app/) (~8.2 million US customers affected).
