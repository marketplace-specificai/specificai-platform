# Support

## Your dedicated Slack channel

The primary support channel is the **dedicated Slack channel your
organization shares with the SpecificAI team** — set up during onboarding.
Use it for anything: installation and upgrade help, platform questions,
feature requests, and incident reports. It is the fastest way to reach the
people who can help.

## Email

If you do not have a Slack channel yet, or Slack is unavailable, email
`support@specific.ai`. Include your platform version (`helm list` shows the
chart version) and, for installation issues, the cloud provider and cluster
type you are deploying to.

## Security vulnerabilities

Do **not** report suspected security vulnerabilities in Slack or regular
email threads. Follow the coordinated disclosure process in
[SECURITY.md](https://github.com/marketplace-specificai/specificai-platform/blob/main/SECURITY.md)
instead, so the report reaches the right people privately.

## What to include in a report

For platform issues, the following shortens every round-trip:

- Chart version (`helm list --namespace <your namespace>`).
- Cloud provider and cluster type (the values template you started from).
- What you did, what you expected, and what happened instead.
- Relevant `kubectl` output — failing pods (`kubectl get pods`), events
  (`kubectl describe pod <pod>`), and logs (`kubectl logs <pod>`).
