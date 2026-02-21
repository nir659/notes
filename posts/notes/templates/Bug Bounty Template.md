BugCrowd templates for Good PoC and Good Report structures.

## Good PoC Examples

| Issue Type | Proof of Concept |
|------------|------------------|
| **Cross-site scripting** | `alert(document.domain)` or `setInterval`alert\\x28document.domain\\x29`` if you have to use backticks. Using `document.domain` instead of `alert(1)` can help avoid reporting XSS bugs in sandbox domains. |
| **Command execution** | <p>Depends on program rules:</p><ul><li>Read (Linux-based): `cat /proc/1/maps`</li><li>Write (Linux-based): `touch /root/your_username`</li><li>Execute (Linux-based): `id`</li></ul> |
| **Code execution** | <p>This involves the manipulation of a web app such that server-side code (e.g. PHP) is executed.</p><ul><li>PHP: `<?php echo 7\\*7; ?>`</li></ul> |
| **SQL injection** | <p>Zero impact</p><ul><li>MySQL and MSSQL: `SELECT @@version`</li><li>Oracle: `SELECT version FROM v$instance;`</li><li>Postgres SQL: `SELECT version()`</li></ul> |
| **Unvalidated redirect** | <ul><li>Set the redirect endpoint to a known safe domain (e.g. google.com), or if looking to demonstrate potential impact, to your own website with an example login screen resembling the target's.</li><li>If the target uses OAuth, you can try to leak the OAuth token to your server to maximise impact.</li></ul> |
| **Information exposure** | Investigate only with the IDs of your own test accounts — do not leverage the issue against other users' data — and describe your full reproduction process in the report. |
| **Cross-site request forgery** | When designing a real-world example, either hide the form (`style="display:none;"`) and make it submit automatically, or design it so that it resembles a component from the target's page. |
| **Server-side request forgery** | <p>The impact of a SSRF bug will vary — a non-exhaustive list of proof of concepts includes:</p><ul><li>reading local files</li><li>obtaining cloud instance metadata</li><li>making requests to internal services (e.g. Redis)</li><li>accessing firewalled databases</li></ul> |
| **Local file read** | Make sure to only retrieve a harmless file. Check the program security policy as a specific file may be designated for testing. |
| **XML external entity processing** | Output random harmless data. |
| **Sub-domain takeover** | Claim the sub-domain discreetly and serve a harmless file on a hidden page. Do not serve content on the index page. |

## Good Report Template

```markdown
# Writeups
# [https://github.com/devanshbatham/Awesome-Bugbounty-Writeups](https://github.com/devanshbatham/Awesome-Bugbounty-Writeups)

# Bug Bounty Report

## Summary
...

## Vulnerability Details
...

## Impact
...

## Proof of Concept
...

## Browsers Verified In
...

## Mitigation
...