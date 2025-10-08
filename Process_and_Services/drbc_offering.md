# Service definition DR/BC Add-on

Our DevOps‑as‑a‑Service offering covers a **base [backup‑and‑restore strategy](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html#backup-and-restore)**:

- encrypted, scheduled backups of in‑scope data stores inside the same cloud account and region
- manual, in‑place restores supported during business hours (or via the standard 24 / 7 escalation path)

This baseline protects against everyday mishaps (bad deploys, accidental deletions) but **does not include outages, cross‑region resilience or coordinated runbook definitions and executions**.

The **Disaster Recovery (DR) Service** is a modular **add-on** to our standard DevOps-as-a-Service offering that fills the gap for customers who require mitigation from large-scale or complex disasters or to meet ISO 27001 / ISO 22301 and similar ISMS requirements.

If you have any additional questions or need further details on deliverables for each step, feel free to let us know. We look forward to helping you implement a robust disaster recovery strategy.

The DR Service is offered in 4 phases:

## Phase 0: Discovery

Before we start the project we spend a short session understanding your business drivers, regulatory context, and existing recovery posture. The outcome is a concise Statement of Work (SoW) that outlines Phase 1.

**Deliverables:**

- **Statement of Work (SoW) for Scoping:** Outlines the high level scope, objectives and target timeline for Phase 1

**Milestones:**

1. SoW delivered to the customer and accepted

## Phase 1: Scoping

In this first step, we collaborate with you to determine the scope:

- **Disaster scenarios and likelihood:** the events you want to recover from and how probable they are
- **Target RTO /  RPO:** the maximum downtime and data‑loss window you can tolerate
- **In‑scope workloads:** applications, data stores, and dependencies we must protect
- [**Recovery strategy**](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html): which pattern (backup & restore, pilot‑light, warm‑standby, or active/active) best fits each workload

With those decisions made, we can produce a clear roadmap that spells out the actions, timeline, and responsibilities required to reach the agreed goal.

**Deliverables:**

- **DRP Scope**: A document that defines precisely which disaster scenarios will be addressed and what workloads, infrastructure and dependencies are in scope, and for each what RTO/RPO's are aimed for and what chosen fail-over strategies are needed for that.
- **Statement of Work (SoW) for Implementation**: A document that outlines the architecture requirements, selected operational addons (phase 3), timing and cost estimates for the cloud costs and the subsequent phases.

**Milestones:**

1. DRP Scope determined and agreed upon by all parties
2. SoW delivered to the customer and accepted

## Phase 2: Implementation

In this phase we implement the necessary infrastructure to support the scope determined in the plan. Once the DR infrastructure is ready, together we create DR runbooks (sometimes called playbooks) that covers the disaster scenario(s). These documents describe a detailed and step by step plan on how to recover each critical workload if a disaster occurs.

Skyscrapers focuses on infrastructure tasks and guides you on relevant application tasks, but both sides have responsibilities in finalizing these procedures.

**Deliverables:**

- **Implemented DR Infrastructure**: The new or updated infrastructure components are deployed in your environment required to meet the scope defined in the SoW adhering to the Skyscrapers best practices.
- **Maintenance runbook**: that defines the cadence, prerequisites and roles for future rehearsals.
- **DR Runbook(s)**: A clear procedure, stored in our shared GitHub repository, covering all steps needed to recover from the disasters identified during the exploration. Together we practice an initial disaster recovery, in an iterative process, to verify and tweak the runbooks’ accuracy and validate the reliability of the created infrastructure until we complete a disaster recovery test successfully. Resulting in a first report that details what was tested, what the RTO/RPO times were, and what further improvements could be made.

**Milestones:**

1. Determine timing and availability of all involved people
2. Delivery of the required infrastructure
3. Delivery of the initial maintenance and DR runbook(s)
4. Delivery of disaster recovery rehearsal

## Phase 3: Operations

This phase includes recurring optional services that ensures the DR environment remains in sync and the runbooks remain clear and actionable:

### Recurring testing & rehearsal add-on

Before relying on any DR plan in a real emergency, it is essential to test it. During this step, we run through rehearsals of the DR procedures to verify and tweak the playbooks’ accuracy and validate the reliability of the created infrastructure. The intensity and frequency of these rehearsals depend on what is determined during the exploration phase.

**Deliverables:**

- **Test/Rehearsal Report**: A written summary of what was tested, how it was performed, and the outcome.
- **Improved Runbook(s)**: In case adjustments were needed we make the needed adjustments to the runbooks so they remain as true as possible.
- **Action Items**: A clear list of improvements or follow-up tasks, assigned either to Skyscrapers or to you, based on the test results to make sure the objectives set forward are met.

### 24/7 DR Standby add-on

Skyscrapers can provide 24/7 coverage specifically for disaster recovery scenarios. While DevOps-as-a-Service includes on-call support for typical production incidents, your DR plan might require a different SLA.

This **DR 24/7** offering ensures that major disruptions, as defined in your disaster scenarios, receive focused attention and swift escalations according to your expectations and needs.

**Deliverables:**

- **DR SLA:** a service level agreement with what we can do in case of the DR scenario and what requirements need to be met to meet this SLA.
- **24/7 On-Call Service for DR Incidents**: A defined escalation path for incidents that meet the threshold of “disaster,” and a dedicated team ready to respond.
- **Recovery Assistance**: Actual work to support, guide, and/or execute recovery steps when a real disaster occurs.

❗Without the 24/7 DR Standby add-on, in the event of a disaster of any type:

1. Skyscrapers cannot make guarantees regarding service availability, service recovery time, data recovery time, or the scope of data that may be recoverable.
2. The existing DevOps-as-a-Service Agreement offers the following limited operational services in the context of DR/BC. During Business Hours:

   - Non-urgent support for questions and minor modifications on the DR/BC setup
   - Support with restoring backups
   - Support with deploying applications to managed platforms
   - Planned execution of DR/BC playbooks

❗This add-on requires the [**Recurring testing & rehearsal add-on**](https://docs.skyscrapers.eu/Process_and_Services/drbc_offering/#recurring-testing-rehearsal-add-on)

## Important Notes

- **Shared Responsibilities**: Recovering infrastructure from a disaster is a shared responsibility. Skyscrapers provides infrastructure and runbook guidance, while you, the customer, remains responsible for identifying which workloads or data must be recovered, along with any application-specific recovery actions. This will be further detailed in your specific DR/BC responsibility model.
- **Pricing & Engagement**: Before starting each phase we will make a budget for you to review before starting the implementation. If you choose the one of the operational add-ons, an additional monthly fee applies for that.
