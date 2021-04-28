# Skyscrapers Support Process

## Priorities

We handle service requests via 3 main categories: Production down, urgent, and non-urgent. Below is the expected response time for each category, what channel to use for it and when to use it:

| Priority        | Production down          | Urgent        |  Non-urgent        |
|-----------------|--------------------------|---------------|--------------------|
| **Scope**           | Only for requests type “Problem” about issues affecting a lot of end-users.  | For requests that are blocking work or are affecting end-users  | For all normal requests that can be handled in a regular flow |
| **Maximum Response time** (to confirm the request)     | < 30 minutes 24/7   | 1 business hour   | Next Business day (unless otherwise agreed)   | 
| **Channel to use** [(more details)](#channels)   | Escalation number  | `@help` on Slack + Github (depending on [request type](#request-types))  |Github |

While the above table sums up how the support process works, here are more details for further reading:

## Channels

### Dedicated 24/7 phone number

- For very urgent requests related to major production events requiring immediate attention, regardless of the time of day.

### Slack

- Short line of communication to Skyscrapers. 
- @help as start of the support process for all Urgent requests and questions about the cooperation and processes (like requests needing special attention).
- General communication with the lead and other colleagues about Non-Urgent requests and other things.

### GitHub Issues

For requests that:

- Require involvement from other parts of the team like the Customer Lead or Engineering, or
- Need further follow-up
- For all Non-Urgent requests

## Request Types

### Usage support

- Practical questions regarding usage of the tools and environments we are providing (Terraform, kubectl, AWS console, etc)

- Slack conversation with `@help` is enough for this type.

### Change Request

- Something needs a change (add, modify, remove) on the existing customer environment and within the scope of the standard technology available.

- Always requires a Github Issue, however, when urgent can be initiated from Slack by pinging `@help`

### Problem

- Something is broken or not behaving as expected. The focus is on solving the immediate problem.

- Requires a Github Issue for following up and further tracking.

- Can be initiated with calling 24/7 number if urgent, or with pinging `@help` on Slack

### Deeper investigation

- Deeper troubleshooting, like debugging of non-urgent problem.

- Requires a Github Issue to be initiated

### Feature request

- We’d like a new functionality or technological component not rolled out today.

- Requires a Github Issue to be initiated.

### Guidance

- Guidance and explanations on architecture, technologies, reference solution, etc.

- Requires a Github Issue to be initiated.

### Other

- Anything else. Initiate with a Github issue or by pinging `@help` on Slack according to [priority](#priorities).
