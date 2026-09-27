# Finding Stale Active Directory Accounts with PowerShell

A stale Active Directory account is an account that still exists in the directory but has not been used for a significant amount of time. This does not automatically mean the account should be deleted. A user may be on long-term leave, an account may belong to a service, or an administrator account may simply be used rarely. However, unused accounts are worth reviewing because forgotten but still-enabled accounts increase the attack surface of a domain.

For this exercise, I used _90 days without a logon_ as the threshold for identifying accounts that may need investigation.
## Useful Active Directory Properties

The `Get-ADUser` cmdlet can retrieve several properties that are useful when reviewing accounts:

- `Enabled` : whether the account is currently enabled
- `LastLogonDate` : a readable representation of the replicated `lastLogonTimestamp`
- `PasswordLastSet` : when the account's password was last changed
- `DistinguishedName` : where the account is located in Active Directory
- `SamAccountName` : the account's logon name

A basic query looks like this:

```powershell

Get-ADUser -Filter * -Properties Enabled, LastLogonDate, PasswordLastSet
```

By default, `Get-ADUser` only returns a limited set of attributes, so the additional properties need to be requested explicitly.

## Finding Accounts With No Logon for 90+ Days

First, I can calculate the cutoff date:

```powershell
$cutoff = (Get-Date).AddDays(-90)
```

Then I can retrieve enabled users whose last recorded logon is older than that date:

```powershell
Get-ADUser -Filter * -Properties LastLogonDate, PasswordLastSet | Where-Object {
        $_.Enabled -eq $true -and
        $_.LastLogonDate -lt $cutoff
    } | Select-Object Name, SamAccountName, LastLogonDate, PasswordLastSet
```

This gives me a much more manageable list than pulling and inspecting every domain user manually.

The output might look something like this:

```text
Name            SamAccountName    LastLogonDate        PasswordLastSet
----            --------------    -------------        ---------------
Test User 01    test.user01       04/15/2026 09:42     02/10/2026 14:11
Test User 02    test.user02       01/27/2026 16:03     12/18/2025 11:26
```

## What About Accounts That Have Never Logged In?

Some accounts may have no `LastLogonDate` at all. For example, an account might have been created but never actually used.

I can include those separately:

``` powershell
Get-ADUser -Filter * -Properties LastLogonDate, PasswordLastSet | Where-Object {
        $_.Enabled -eq $true -and
        $null -eq $_.LastLogonDate
    } | Select-Object Name, SamAccountName, LastLogonDate, PasswordLastSet
```

A never-used enabled account can be interesting during an audit, but context matters. It may simply be a newly created account waiting for an employee to start...

## Combining Both Checks

To find both accounts that have been inactive for more than 90 days and accounts that have never logged in:

```powershell
$cutoff = (Get-Date).AddDays(-90)

Get-ADUser -Filter * -Properties LastLogonDate, PasswordLastSet | Where-Object {
        $_.Enabled -eq $true -and
        ($null -eq $_.LastLogonDate -or $_.LastLogonDate -lt $cutoff)
    } | Select-Object Name, SamAccountName, LastLogonDate, PasswordLastSet | Sort-Object LastLogonDate
```

Sorting by `LastLogonDate` makes the oldest accounts easier to spot.

## Exporting the Results

For a larger environment, reading the results directly in PowerShell is not especially convenient. The results can instead be exported to CSV:

```powershell
$cutoff = (Get-Date).AddDays(-90)

Get-ADUser -Filter * -Properties LastLogonDate, PasswordLastSet | Where-Object {
        $_.Enabled -eq $true -and
        ($null -eq $_.LastLogonDate -or $_.LastLogonDate -lt $cutoff)
    } | Select-Object Name, SamAccountName, LastLogonDate, PasswordLastSet | Sort-Object LastLogonDate | Export-Csv ".\AD_90plus_inactive.csv" -NoTypeInformation -Encoding UTF8
```

This creates a file called:

```text
AD_90plus_inactive.csv
```

I can then review the CSV in Excel or another analysis tool.

## A Note About LastLogonDate

One important thing is that `LastLogonDate` should not be treated as an exact forensic timestamp; it is simply derived from Active Directory's replicated `lastLogonTimestamp` attribute, which is not updated on every logon.[^1] It is useful for when you'd like to find which accounts have apparently not been used for around 90 days, but not for determining what was the exact last time this user authenticated.

For an inactivity review like this exercise, this distinction is usually acceptable because the threshold is measured in months rather than hours.

## Why I Would Not Automatically Disable the Results

Honestly, it _is_ tempting to take the output of this query and immediately disable everything in it.

That would be a bad idea.

There are some questions that need to be addressed before making any changes:

-  Does the account belong to a current employee?
-  Is the employee on leave?
-  Is this actually a service or application account?
-  Is it an administrative account that is intentionally used only occasionally?
-  Has the employee left the organization?
-  Does the account still have access to VPN, email, file shares, or other systems?
-  Is there an established offboarding process for disabling accounts?

"_This account has not logged in for 90 days_" is not that interesting of a finding. A genuinely interesting finding would be closer to: 
	"_This account has not logged in for 90 days, remains enabled, and there is no documented business reason for it to remain active._"

## Security Relevance

Stale accounts matter because every enabled identity represents another possible way into an environment.

The risk becomes more significant when a stale account also has:

-  administrative privileges,
-  remote access,
-  access to sensitive systems,
-  a weak, reused, or potentially compromised password,
-  no MFA,
-  or an owner who has already left the organization.

This is why account inventories should be reviewed together with permissions and employee status rather than in isolation.

## What I Learned

The main takeaway from this exercise is that PowerShell makes finding potentially inactive accounts easy. The tricky part is _correctly_ deciding what the results mean.

`Get-ADUser` can tell me whether an account is enabled, when it last logged in (approximately), and when its password was last changed. It sadly cannot tell me whether the account still has a legitimate business purpose—that requires verification. If only it could.


[^1]: Microsoft, [MS-ADLS: `lastLogonTimestamp` Attribute](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adls/ba6d87f9-7023-4ccd-9d0e-dd7b53865db5)