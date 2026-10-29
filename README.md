# Lab 1: Entra ID Users, Groups and Least-Privilege Admin Roles

**Platform:** Microsoft Entra ID (free tier)

**Track:** IAM / Help Desk home lab

**Exam alignment:** SC-300, Implement and manage user identities

* * *

## 1. Summary

I set up our employee accounts and made sure each person had only the access they need to do their job. For example, Papa can reset Gabriel's password but can't reset Aiden's, because Aiden has a higher level of control over the company's systems. This means that if an attacker broke into Papa's account, they would have limited access and could not take over the company's most important accounts.

* * *

## 2. The scenario

**Company:** Calgary Energy Service Co

**Location:** Calgary, Alberta

**Industry:** Energy

| Name | Title | Department |
| --- | --- | --- |
| Ebenezer Aibi | Chief Executive Officer | Management and Administration |
| Michelle Wurtz | Finance Manager | Management and Administration |
| Ebenezer Mensah | Human Resources | Management and Administration |
| Papa Aibi | IT and Help Desk Technician | IT and Technical Support |
| Aiden Essah | System Administrator | IT and Technical Support |
| Jamel Aidoo | Field Operations Manager | Field Operations |
| Augustine Biney | Energy Engineer | Field Operations |
| Gabriel Addae | Field Technician | Field Operations |
| Temp Accountant | Temporary Accountant (contractor) | Management and Administration |

* * *

## 3. Design decisions

### Department groups vs access groups

One of the main lessons from this lab was that department membership and access to resources should not be represented by the same group. A department group answers *who is this person?* An access group answers *what can this person open?*

The two often start with the same members, but they drift apart as soon as someone outside a department needs access.

**Department groups**

| Group | Members |
| --- | --- |
| Management-Admin | Ebenezer Aibi, Michelle Wurtz, Ebenezer Mensah |
| IT-Technical-Support | Papa Aibi, Aiden Essah |
| Field-Operations | Jamel Aidoo, Augustine Biney, Gabriel Addae |

### Field data

| Group | Members | Access |
| --- | --- | --- |
| Field-Data-Edit | Jamel Aidoo, Augustine Biney, Gabriel Addae | Create and edit reports, use field tools |
| Field-Data-Read | Aiden Essah (contractors to be added as they join) | View field and site reports only |

Aiden works in IT but needs to read equipment logs to troubleshoot a field tablet. A future contractor might need site reports for three months without access to the tools. Neither belongs in the Field Operations department, so access is granted through Field-Data-Read instead. Access can change without changing anyone's department.

### Finance-Payroll

| Group | Members | Owner | Planned backup |
| --- | --- | --- | --- |
| Finance-Payroll | Michelle Wurtz, Temp Accountant | Michelle Wurtz | Ebenezer Aibi (planned, via PIM) |

Michelle normally manages payroll, which includes salaries and company banking details. Making her the only person with access created a single point of failure: if she were off sick on payday, nobody could run payroll. Ebenezer Aibi, the CEO, is the named backup.

**Planned improvement, PIM for Groups:**

| Person | Access | How |
| --- | --- | --- |
| Michelle Wurtz | Normal Finance-Payroll access | Permanent business requirement |
| Ebenezer Aibi | Backup Finance-Payroll access | Eligible through PIM when Michelle is unavailable |

With PIM, Ebenezer would hold no payroll access day to day. He would activate it for a few hours when needed, and every request would be logged. This needs a premium licence, so it is planned for the governance phase of the lab.

### Employee records

| Group | Member | Owner | Permission | Reason |
| --- | --- | --- | --- | --- |
| Employee-Records-Edit | Ebenezer Mensah | Ebenezer Mensah | Read and edit | Human Resources owns and maintains employee records |
| Employee-Records-Read | Michelle Wurtz | Ebenezer Mensah | Read only | Finance needs salary details for payroll |

Two groups are needed because a group gives every member the same level of access. One group with mixed permissions would either let Michelle edit records or stop Ebenezer Mensah from doing his job.

Ownership follows responsibility, not membership: Ebenezer Mensah owns Employee-Records-Read even though he is not in it, because he decides who may read employee records.

### IT tools

| Group | Members | Owner | Access |
| --- | --- | --- | --- |
| IT-Admin-Tools | Papa Aibi, Aiden Essah | Aiden Essah | IT management tools and admin documentation |

### Admin roles

| Person | Role | Why |
| --- | --- | --- |
| Papa Aibi | Password Administrator | Resets passwords for ordinary users only, the narrowest role for his help desk work |
| Aiden Essah | User Administrator | Creates and manages users and groups, and can reset passwords for lower admins such as Papa |

Nobody was given Global Administrator. It controls the whole tenant, and the tasks above don't need it. Each admin role can reset passwords only for people at or below its own level, which prevents privilege escalation: a lower admin resetting a higher admin's password to take over their access.

* * *

## 4. What I built

* 9 users with job title, department, company name and usage location (Canada)
* 9 security groups with assigned membership: 3 department groups and 6 access groups, each with a description and, for access groups, a named owner
* 2 least-privilege admin role assignments
* A tenant admin account clearly labelled **(ADMIN)**, kept separate from all business groups

| Screenshot | Shows |
| --- | --- |
| 01-users.png | All users, with department column |
| 02-groups.png |  All ten accounts: nine staff plus the labelled admin account|
| 03-finance-payroll-members.png | An access group's members |
| 04-papa-roles.png | Papa: Password Administrator |
| 05-aiden-roles.png | Aiden: User Administrator |

*Sign-in names, personal email addresses and temporary passwords are blurred.*

* * *

## 5. Testing

| Test | My prediction | Result |
| --- | --- | --- |
| Papa resets Gabriel's password | Works | Works (after the fix in section 6) |
| Papa resets Aiden's password | Works | Blocked: Papa cannot reset a higher admin |
| Michelle adds a member to Finance-Payroll, which she owns | Works | Works, with no admin role |
| Michelle adds a member to Employee-Records-Read, which she doesn't own | Blocked | Blocked: the Add members option isn't available to her |

My prediction for Aiden was wrong, and the test showed why: role hierarchy, not just the role name, decides whose password an admin can reset.

| Screenshot | Shows |
| --- | --- |
| 07-gabriel-reset-success.png | Papa resets Gabriel's password |
| 08-aiden-reset-blocked.png | Papa is blocked from resetting Aiden's password |
| 09-owner-adds-member.png | Michelle adds Temp Accountant to Finance-Payroll |

* * *

## 6. What broke and how I fixed it

### Issue 1: Role names typed into the Job Title field

**What happened**

While assigning administrative roles in my Calgary Energy Service Co tenant, I confused the Job Title field with role assignment. Instead of assigning the roles through Assigned roles, I typed the role names into the users' Job Title fields. Aiden Essah's job title became "User Administrator", but no role was actually assigned.

**How I noticed**

I signed in as Papa Aibi, who was meant to be a Password Administrator, and tried to reset Gabriel Addae's and Aiden Essah's passwords. Both resets were blocked. The Gabriel result was the red flag: a Password Administrator should always be able to reset an ordinary user with no admin role, so something in the setup was wrong.

**How I found the cause**

Aiden's profile showed 0 assigned roles. But I was viewing it from Papa's account, which may not be allowed to see role assignments, so I didn't trust that evidence. I signed in with my administrator account, which confirmed that neither Papa nor Aiden had any role. Opening Edit properties on Aiden's profile showed the cause: "User Administrator" was sitting in his Job Title field.

**How I fixed it**

* Restored the original job titles: Papa Aibi as IT and Help Desk Technician, Aiden Essah as System Administrator
* Assigned the real roles through Assigned roles: Password Administrator for Papa, User Administrator for Aiden
* Confirmed each profile showed one assigned role
* Signed Papa out and back in, so his session picked up the new role
* Retested: Papa's reset of Gabriel succeeded, and his reset of Aiden was blocked, exactly as designed

**Lesson:** a job title is just a label. It grants no access.

*Screenshot: 06-job-title-mistake.png*

### Issue 2: Admin account in a business group, and a missing CEO account

**What happened**

I filled in the user form for the CEO, Ebenezer Aibi, but the account was never actually created, most likely because I didn't click Create on the review screen. My own Global Administrator account also uses the name Ebenezer Aibi. Later, when I added "Ebenezer Aibi" to the Management-Admin group, I picked the only account with that name, which was my admin account.

This was a security problem. The account that controls the entire tenant was sitting in an ordinary business group, so any access or policy aimed at that group would also have applied to it. It also meant the most powerful account in the tenant was being treated as an everyday business account, exposing it to far more risk than it needs.

**How I noticed**

While checking my screenshots for this write-up, the user count didn't add up. The list showed 8 users, but it should have shown 9: eight employees plus my admin account. Only one Ebenezer Aibi appeared, and its sign-in name contained #EXT#, not the company sign-in name I had given the CEO.

**How I found the cause**

I opened that account's profile. Three details showed it was my administrator account, not the CEO:

* Its sign-in name contained **#EXT#**, which marks an identity from outside the tenant
* It was created in **April**, months before I created the company's users
* It had **one assigned role**: Global Administrator, while the CEO was designed with none

Its profile also showed **one group membership**. Clicking it showed that the account was a member of Management-Admin.

**How I fixed it**

* Created the CEO account, Ebenezer Aibi, with the correct sign-in name, job title, department and usage location
* Added the new CEO account to Management-Admin **before** removing the admin account, so the group was never without its CEO
* **Removed**, not deleted, my admin account from the group; deleting it would have locked me out of my own tenant
* Renamed my admin account to **Ebenezer Aibi (ADMIN)** so it can't be confused with the CEO again
* Confirmed the user list now showed 9 users

**Lesson:** a highly privileged administrator account should be kept separate from everyday business accounts and groups, and clearly named so it can't be mistaken for anyone else.

* * *

## 7. What I learned

**Least privilege limits the damage when something goes wrong.**

Giving each person only the access their job needs means that if an account is compromised, the attacker gets only what that account could do. My testing showed it in practice: Papa, a Password Administrator, could reset an ordinary user's password but was blocked from resetting Aiden's, because Aiden holds a higher admin role. If Papa's account were stolen, the damage would stop there.

**Job titles and permissions are different things.**

A user can be called "System Administrator" or "Password Administrator", but the title gives them no access at all. Roles have to be assigned separately. I learned this by making the mistake myself, then troubleshooting why the permissions weren't working.

**Department groups and access groups answer different questions.**

A department group says who someone is; an access group says what they can open. They often start with the same members, but they drift apart as soon as someone outside a department needs access, like Aiden in IT reading field reports. Designing access around what people need, rather than where they sit, kept sensitive data such as payroll away from people who didn't need it.

**Privileged accounts need to be kept separate and clearly named.**

My Global Administrator account shared the CEO's name, and when the CEO account was never actually created, I added my admin account to a business group by mistake. That showed me why administrator accounts should be clearly labelled and kept out of everyday business groups.

* * *

## 8. SC-300 objectives covered

From the SC-300 study guide, *Implement and manage user identities*:

* Create, configure and manage users
* Create, configure and manage groups
* Configure and manage built-in Microsoft Entra roles
* Evaluate effective permissions for Microsoft Entra roles
* Configure tenant properties, user settings and group settings

Explored, or planned for later phases:

* Tenant-wide MFA settings: observed through security defaults during first sign-in
* PIM for Groups: designed for the payroll backup, to be built in the governance phase
