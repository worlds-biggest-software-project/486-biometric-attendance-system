# Biometric Attendance System

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, privacy-first biometric time-and-attendance platform that eliminates buddy punching and payroll fraud while giving organisations full control over how biometric data is stored, processed, and deleted.

Biometric Attendance System is a self-hostable time-tracking platform that uses facial recognition and fingerprint verification to bind every clock event to a verified individual. It targets HR teams, operations managers, and payroll administrators at mid-to-large organisations who need fraud-proof attendance records without surrendering employee biometric data to a third-party cloud. The project addresses the estimated 2-8% of gross payroll lost annually to time theft and manual attendance errors.

---

## Why Biometric Attendance System?

- **No open-source product matches SaaS polish.** TimeTrex is the only open-source option with a payroll engine, but its UI is dated and hardware terminal integration is limited. Commercial SaaS tools like Jibble and Truein offer modern interfaces but lock organisations into proprietary platforms.
- **Privacy compliance is bolted on, not built in.** ZKTeco lacks consent management and GDPR deletion tooling entirely. Suprema and Invixium offer no automated multi-jurisdictional consent workflows. With BIPA class-action lawsuits multiplying and GDPR Article 9 imposing strict requirements, organisations need compliance as a core feature, not a manual afterthought.
- **No product offers AI-native anomaly detection.** Every incumbent relies on static threshold rules to flag attendance exceptions. None use ML to detect buddy punching clusters, systematic early departures, or overtime spikes that evade simple rules.
- **Hardware vendor lock-in fragments the market.** ZKTeco software works best with ZKTeco devices. Suprema BioStar is tightly coupled to Suprema hardware. SaaS-only tools like Connecteam skip hardware entirely and cannot support environments that need dedicated terminals. No device-agnostic open platform bridges all three worlds under a single API.
- **Enterprise pricing excludes mid-market.** Invixium and Suprema target large enterprises with high per-terminal costs. Connecteam restricts SSO, 2FA, and full API access to its Enterprise tier. Truein and Keka focus primarily on the Indian market with limited Western payroll connectors.

---

## Key Features

### Biometric Capture and Anti-Spoofing

- Multi-modal biometric authentication: facial recognition (2D/3D with liveness detection), capacitive fingerprint, and extensible support for palm vein and iris scanning
- Anti-spoofing via IR-based and depth-sensing liveness checks to detect printed photos, silicone fingerprints, and video replay attacks
- On-device template storage using irreversible, cancellable biometric representations -- never raw images
- Geo-fenced mobile check-in with GPS validation and device biometrics (Face ID / fingerprint) for field workers

### Workforce Management

- Shift scheduling with configurable work rules, overtime calculation, and exception handling
- Manager exception dashboard surfacing late arrivals, absences, early departures, and out-of-schedule clock-ins
- Leave and PTO module integrated with attendance calculations
- Employee self-service portal for viewing history, requesting leave, and submitting corrections

### Privacy and Compliance

- Informed consent capture workflow before biometric enrolment with opt-out handling
- Automated retention and deletion schedules configurable per jurisdiction (GDPR, BIPA, PDPA, POPIA)
- Cancellable biometrics scheme: if a template is compromised, it can be revoked and re-enrolled with a different transformation
- Configurable data residency to keep EU data in EU regions

### Integration and Deployment

- Payroll export via REST API connectors and CSV for ADP, Paychex, SAP SuccessFactors, Xero, and QuickBooks
- Open protocol adapters for ZKTeco and Suprema hardware terminals, plus mobile-only deployment with no hardware dependency
- Offline-first architecture: terminals authenticate locally and queue clock events for sync on reconnection
- Immutable, timestamped audit log of every clock event, consent action, and administrative change

### Analytics and Intelligence

- AI anomaly detection flagging unusual attendance patterns (buddy punching clusters, overtime spikes, systematic schedule violations) without manual threshold configuration
- Predictive absence management using historical attendance and calendar patterns
- Natural-language attendance reporting allowing HR to query data conversationally

---

## AI-Native Advantage

Unlike incumbents that rely on static rules and manual review, this system uses ML models trained on attendance patterns to surface fraud and anomalies that threshold-based approaches miss -- such as co-located buddy punching, shift swaps that violate contract limits, or gradually shifting clock-in times that stay just within policy. An LLM-powered reporting interface lets HR teams ask questions in plain English ("Show me departments with rising absenteeism this quarter") instead of building custom report filters. On-device ML models for presentation attack detection continuously improve from flagged spoofing attempts, hardening liveness checks over time.

---

## Tech Stack & Deployment

The platform supports three deployment modes: self-hosted on-premises for organisations with strict data sovereignty requirements, cloud-hosted SaaS, and hybrid (cloud management plane with on-premises biometric processing). Biometric matching happens on-device by default, with templates encrypted at rest using per-device keys and never transmitted to the cloud without homomorphic protection. A unified device integration API abstracts across ZKTeco, Suprema, eSSL, and mobile device sensors, eliminating vendor lock-in. The initial release targets mobile-first biometric capture (smartphone cameras and fingerprint sensors), with hardware terminal adapters following in v1.1.

---

## Market Context

The global biometric system market is projected to grow from $53.22 billion in 2025 to $95.14 billion by 2030 (12.3% CAGR), with enterprise time-and-attendance as a mature but actively innovating segment. Nucleus Research found that automating attendance with biometrics delivers ROI in under nine months. Current pricing ranges from free tiers with limited features (Jibble) to $1.50+/user/month SaaS (Truein) to high-cost enterprise hardware bundles (Invixium, Suprema), leaving a gap for an open-source solution that combines enterprise-grade accuracy with transparent, self-hostable deployment.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
