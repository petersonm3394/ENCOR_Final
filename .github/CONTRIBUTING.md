# Contributing Guidelines

Thanks for contributing. This project is designed for a student lab environment, so we prioritize clarity, repeatability, and safe defaults over complexity.

---

## 1) Scope of Contributions

We welcome contributions in these areas:

- Documentation improvements (typos, clarifications, screenshots, troubleshooting steps)
- Safer defaults (locking down security group rules, hardening configs)
- Reliability fixes (better Promtail parsing, more robust rsyslog rules, cleaner Docker Compose setup)
- Additional dashboards/panels (Grafana JSON exports, example queries)
- Optional enhancements that remain low-cost (within AWS free tier where possible)

Please avoid:
- Changes that require paid SaaS products
- Changes that require inbound connectivity to the home lab (unless clearly marked as optional)
- Overly large “framework” additions that make the lab harder to run

---

## 2) Development Standards

### Documentation
- Write instructions so a new student can follow them without prior context.
- Use consistent placeholders:
  - `<EC2_PUBLIC_IP>`
  - `<YOUR_HOME_PUBLIC_IP>`
  - `<LAN_INTERFACE>`
  - `<TEST_INTERFACE>`
- Include verification commands after major steps (e.g., `ss -lunp`, `tail -f`, `docker ps`).

### Security defaults
- Security Group rules must default to **least privilege**:
  - Prefer `<YOUR_HOME_PUBLIC_IP>/32` over `0.0.0.0/0`
- If you include an “open to the world” rule, label it explicitly as **temporary** and include a note to tighten it.

### Compose and config changes
- Keep the stack runnable with:
  ```bash
  docker compose up -d
