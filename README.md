# Android Security Notes

Short, practical write-ups of security issues I find and fix **in my own apps** while learning mobile security. Each note explains the bug, its real impact, the fix, and a check you can run on your own project. Code samples are simplified and use generic names.

All testing is done only on my own apps and devices.

| # | Topic | Category |
|---|---|---|
| 01 | [Exported debug receivers: a backdoor you add yourself](01-exported-debug-receivers.md) | CWE-926 Improper export of Android components |
| 02 | [Supabase: one table without RLS, and RLS that protects rows but not columns](02-supabase-rls-is-not-column-security.md) | CWE-862 Missing authorization · CWE-915 Mass assignment |
| 03 | [A backup file is untrusted input too: one integer freezes the app, one date breaks a screen for good](03-a-backup-file-is-untrusted-input.md) | CWE-20 Improper input validation · CWE-190 Integer overflow · CWE-835 Infinite loop |
