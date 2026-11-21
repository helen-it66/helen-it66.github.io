---
layout: post
title: "Building a Company in Active Directory: OUs, Users and Groups with PowerShell"
---

In my [last post]({{ site.baseurl }}{% post_url 2026-11-16-building-a-domain-controller %}), I promoted my Windows Server 2025 VM (DC01) to a domain controller for `helenlab.local`. An empty domain isn't much use, so this time I gave it something to manage: a small fictional company called **Northfield Supply Co.**, with departments, employees, security groups and a separate admin account for me.

## The plan

Before creating anything, I sketched out how the company should be organized. Organizational Units (OUs) work like folders in Active Directory. They keep things tidy and, later on, let me apply Group Policy to specific departments.

```
helenlab.local
├── Lab Users
│   ├── Sales
│   ├── HR
│   ├── Finance
│   ├── IT
│   └── Disabled Users
├── Lab Computers
└── Lab Groups
```

The company has 13 people across four departments, plus one former employee whose account is disabled. That account will be useful later for practising offboarding.

| Department | Users |
|---|---|
| Sales | Sales Manager, 2 Account Executives, Sales Coordinator |
| HR | HR Manager, HR Generalist, Recruiter |
| Finance | Finance Manager, Accountant, Payroll Specialist |
| IT | Me (IT Administrator), Help Desk Technician |
| Disabled Users | 1 former Sales rep |

## Security groups

Permissions in a real company are given to **groups**, not individual people. When someone joins or leaves, you change their group membership instead of editing permissions on every folder. I created these Global Security groups in the `Lab Groups` OU:

| Group | Purpose |
|---|---|
| Sales-Staff, HR-Staff, Finance-Staff, IT-Staff | Access to each department's shared folder |
| Managers | Access to a management-only folder |
| Helpdesk-Admins | Password resets and account unlocks |
| All-Staff | Company-wide share and policies |

## Creating everything with PowerShell

I created a few users by hand in **Active Directory Users and Computers** to learn the interface, then used PowerShell for the rest. Typing 13 users into the same form gets slow, and scripting is how admins handle this at scale.

First, two variables so I didn't have to repeat myself:

```powershell
$dc = "DC=helenlab,DC=local"
$pw = Read-Host "Starting password" -AsSecureString
```

`$dc` is my domain name written the way Active Directory describes locations. `$pw` holds the starting password as a SecureString, because `New-ADUser` won't accept a plain text password. (For this post I've swapped my lab password for a prompt, so it isn't published.)

Then the OUs:

```powershell
New-ADOrganizationalUnit "Lab Users" -Path $dc
"Sales","HR","Finance","IT","Disabled Users" | ForEach-Object {
  New-ADOrganizationalUnit $_ -Path "OU=Lab Users,$dc" }
```

The first line creates the parent OU. The second sends a list of five names through a pipe (`|`), and `ForEach-Object` runs the command once for each one, with `$_` standing for the current name. One line creates five OUs. This "list, pipe, loop" pattern shows up all over PowerShell.

The users were created the same way: a list of names, usernames, departments and titles, with a loop that runs `New-ADUser` for each person, places them in their department's OU, adds them to their department group and All-Staff, and sets **User must change password at next logon**. Everyone starts with the same temporary password, but it only works once.

## Problem: half my users were "missing"

**Symptom:** After running the script and refreshing Active Directory Users and Computers, the Sales OU looked empty, and so did my own account's folder.

**Investigation:** Instead of guessing, I asked Active Directory directly where every user was:

```powershell
Get-ADUser -Filter * | Select Name, DistinguishedName
```

All the users existed. The DistinguishedName column showed the real problem: the Sales users and my admin account were in `OU=Lab User`, while everyone else was in `OU=Lab Users`. One missing "s".

**Cause:** I'd created a **Lab User** OU by hand earlier, so I had two nearly identical OUs, and accounts had been split between them.

**Fix:**
1. Moved the users from **Lab User** into the matching OUs under **Lab Users**.
2. Turned on **View → Advanced Features**, unchecked **Protect object from accidental deletion** on the extra OU, and deleted it.
3. Ran the same `Get-ADUser` command again to confirm every account showed `OU=Lab Users`.

**Lesson:** When something in the GUI looks wrong, check it from PowerShell. The DistinguishedName gives the exact location of an object, and a near-duplicate name like this would silently break Group Policy and searches later.

## Using a separate admin account

Until now I'd been logging in as the built-in Administrator. Real companies avoid that. Each admin gets a personal account, so actions can be traced to a person, and the built-in account is kept for emergencies.

I created my own account in the IT OU and added it to **Domain Admins**, **IT-Staff** and **Helpdesk-Admins**. One thing I learned: built-in groups like Domain Admins live in the default **Users** container and should stay there. You don't need to move a group into your own OUs to add people to it.

Group changes only apply when you next log in, so I signed out and signed back in as `HELENLAB\` plus my new username, then confirmed I could still create and edit users.

## What I learned

- **Assign permissions to groups, not people.** It makes onboarding and offboarding a matter of changing group membership.
- **PowerShell is faster and more reliable than clicking** for anything repeated, and reading a script line by line taught me as much as running it.
- **Check the real data when the GUI is confusing.** `Get-ADUser` showed in seconds what clicking around couldn't.
- **Naming consistency matters.** One letter created a duplicate OU that would have caused problems later.

## What's next

I took a snapshot of DC01 (`Users created`) so I can always roll back to this point. Next, I'll create shared folders for each department, lock them down with these security groups, and map them as drives with Group Policy. That's where the practice help desk tickets start.
