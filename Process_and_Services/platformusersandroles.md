# DevOps-as-a-Service: Platform Users and User Roles

## What are Platform Users?

Our DevOps-as-a-Service cooperations all start with the foundational construct of a Platform User:

> “Platform User” is any employee of, contractor of or person related to Customer who may be requesting support from Skyscrapers directly or indirectly, or may be accessing, deploying or maintaining application code through any means possible (UI, API, CI, etc) to platforms managed by Skyscrapers, or may be using or consuming cloud provider services within the cloud accounts managed by Skyscrapers.
> 

This definition is part of your agreement with Skyscrapers.

<aside>
ℹ️ As a customer you are required to keep us up to date on changes in your team, the Platform Users and User Roles.

</aside>

## What are User Roles?

Each Platform User is assigned a User Role. 

Some of the Platform Roles can also be held by non-Platform Users. For example the Contract Owner could be a person not actively using any of the services provided by Skyscrapers.

## Why User Roles?

At Skyscrapers, User Roles are a way to categorise the people we work with on the customer side. It allows us to:

- Better align the delivery of the services to different needs depending on the role people have
- Helps Skyscrapers to understand your organisation
- Set and manage expectations on both sides
- Allows for creating standardised processes and tailored services for each role
- Price differentiation aligned with your specific needs

There is a strong relation between User Roles and the Skyscrapers Responsibility Model.

## User Roles

|  | Platform Lead | Supported Platform User | General Platform User | Contract Owner (*) | Compliance Lead (*) |
| --- | --- | --- | --- | --- | --- |
| Tl;Dr | DevOps-oriented or Developer oriented people we work with closely and overseeing all work between SkS and your own organisation. They give access to the full context on your side. | Developers or related profiles that get access to our support channels. | Developers or related profiles that indirectly work with SkS or on environments managed by SkS, without direct support. | Person that has contractual decision powers and oversight. Can be anybody. | First contact for security/compliance events and questions in both directions |
| Counts as Platform User | Yes | Yes | Yes | No | No |
| Access to: |  |  |  |  |  |
| VPN/AWS/Cluster access | Y | Y | Y | - | - |
| GitHub Issues  | Y, incl. Platform Components requests | Y, Support/advice only | - | Y | - |
| GitHub Repo  | Y | Y | - | Y | - |
| Slack dedicated channel | Y | Y | - | - | - |
| 24/7  | Y | Y | - | - | Y, for security and data incidents |
| Customer specific documentation on GitHub  | Y | Y | - | Y | - |
| Pull Request creation  | Y | Y, Vetted by the Platform Lead first | - | - | - |
| https://docs.skyscrapers.eu/ |  | Y | Y | Y | - |
| Responsibilities | Determine priorities/urgency
Add context where needed
Keep oversight on all support requested from Customer side | - |  | Budget, SLA decisions
Oversee quality of service delivery | DPO and compliance in the cooperation |
| Activities we organise: |  |  |  |  |  |
| Support | Y | Y | - | - | - |
| Trainings | Y | Y | - | - | - |
| Expert Sparring | Y | Y | - | - | - |
| Announcement Updates | Y | Y | - | - | - |
| Status calls | Y | - | - | Y | - |
| Account calls | Y | - | - | Y | - |
| Notes: | Usually 1 person per team |  |  | *: Can also be any of the Platform Users | Required for GDPR
*: Can also be any of the Platform Users |

## Legacy Roles

Before 2024 and depending on the contract you may have, the following roles may have been defined before:

- **Platform User**: didn’t have differentiation between roles
- **Customer Dev Lead**: similar to the Platform Lead
- **Contract Owner**: similar to the current Contract Owner

Over the next few months, your Skyscrapers Customer Lead will work with you in mapping your Platform Users to the new model.

## Why do we use the Platform Users concept?

Our DevOps-as-a-Service cooperations all start with the foundational construct of a Platform User, which is any employee, contractor, or representative of the Customer ...

- who requests support from Skyscrapers (directly or indirectly).
- that accesses, deploys, or maintains application code in platforms managed by Skyscrapers (via UI, API, CI, etc.).
- that uses or accesses cloud provider services (infrastructure) in accounts managed by Skyscrapers.

ℹ️ This definition is part of your agreement with Skyscrapers. As a customer you are required to keep us up to date on changes in your team, the Platform Users and User Roles.

### **Platform Users as pricing model?**

Just like any other company we want to be fairly compensated for the value we deliver, ensuring a sustainable relationship with our customers. Over time, we’ve tried several pricing approaches: from pay-for-time models to fees based on a percentage of cloud costs. Unfortunately, none of these fully aligned our incentives with the interests of our clients.

By pricing based on “Platform Users,” we believe we’ve found a model that’s both fair and aligned.

### What value do we deliver

We build and maintain cloud platforms, handle operational responsibilities, support developers, offer strategic advice, and share in the operational risk (on the platform level). The value of these activities scales with the complexity of your applications and the size of your team.

### Platform Users as a proxy for value

In most digital product companies (like SaaS businesses), there’s a relatively constant ratio of developers and platform engineers (DevOps, SREs, cloud engineers, etc.). 

As your team grows, more developers (”Platform Users”) contribute code, run CI/CD pipelines, and require platform stability, security, and scalability. Each additional developer indirectly increases the demands placed on the platform and, in turn, the platform engineers (support, ops, technology, etc). Thus, the number of Platform Users serves as a reliable proxy parameter for the complexity, support needs, and operational overhead we manage on your behalf.

By using Platform Users as our pricing foundation, we create a fair, transparent, and growth-oriented model that reflects the real value we bring to your team and business. 

### Additional benefits

- **Predictable Costs:** You can forecast and budget the DevOps-as-a-Service costs based on developer team growth, avoiding unexpected expenses for the foundational services.
- **Scalable Pricing:** As your company grows, DevOps-as-a-Service naturally scales with it, so you always pay proportionally to what you need and value brought.
- **Aligned Incentives:** We focus on long-term value and efficiency, not on increasing hours or cloud spend.
- **Sustainable Innovation:** A stable, predictable revenue stream allows us to invest continuously in improving our services, tools, and practices to serve you better.

### FAQ

**What if my developers don’t need to interact with the Skyscrapers team and only push code to the platform through their CI/CD. Does that still count as a Platform User?**

Even if a developer never directly contacts Skyscrapers, their deployments, code changes, and usage of the platform’s infrastructure still generate operational tasks, risks and responsibility. These scale with the number of developers, making every developer a relevant factor to get a fair price for our services. 

Consider a platform where only 2 developers push code to vs a platform where 20 developers push code to. Do you think more will be asked from the platform team in that situation?

**What if certain developers work only part-time, are not pure developers, etc? How is that counted?**

We understand that team compositions vary. Part-time developers, hybrid roles, or contributors who rarely interact with the platform may warrant special consideration. Simply discuss these scenarios with your Customer Lead or sales contact, and we’ll work with you to find a fair arrangement.

**How does Skyscrapers know how many Platform Users should be invoiced?**

We trust you to report changes in team size as they occur and at least each quarter. From time to time, we might ask you to confirm the number of Platform Users to ensure accurate billing. This open, trust-based approach allows us to keep costs transparent and fair for everyone.
