# Skyscrapers Support Process

## Priorities

We handle service requests via 3 main categories: Production down, urgent, and non-urgent. Below is the expected response time for each category, what channel to use for it and when to use it:

|                           |                                         Production down                                         |                             Urgent                             |                        Normal priority                        |
| :-----------------------: | :---------------------------------------------------------------------------------------------: | :------------------------------------------------------------: | :-----------------------------------------------------------: |
|         **Scope**         | Only for requests type “**Problem/Troubleshooting**” about issues affecting a lot of end-users. | For requests that are blocking work or are affecting end-users | For all normal requests that can be handled in a regular flow |
| **Maximum response time** |                                        < 30 minutes 24/7                                        |                        1 business hour                         |          Next Business day (unless otherwise agreed)          |
|    **Channel to use**     |                                        Escalation number                                        |         Log Github issue and `@help` us  through Slack         |         Log Github issue and we will get back to you          |

## Request types

Classifying requests allows to better collect the information beforehand, leading to a higher quality response.

| Type                     | What?                                                                                                                                  |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| Add/Remove User          | A user needs to have access or a user's access needs to be revoked.                                                                    |
| Change Request           | Something needs a change (add, modify, remove) on the existing customer environment. This can be a configuration change or technology. |
| Guidance & usage support | Practical questions, guidance and explanations on architecture, technologies, reference solution, tools, etc                           |
| On-call                  | Messages received during on-call                                                                                                       |
| Problem/Troubleshooting  | Something is broken or not behaving as expected. Troubleshooting and investigation is needed.                                          |
| Other                    | Anything else                                                                                                                          |

## Communication channels

We maintain multiple channels to interact with us. Each has its strengths, depending on the type of request and priority.

| Channels                        |                                                                                                                                                                                                                                                                                                                                                 |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **GitHub Issues**               | <ul> <li>Require involvement from other parts of the team like Expert or Platform</li> <li>For planning and pushing work to us. </li><li>Need further follow-up.</li><li>For all requests with a "Urgent" or "Normal" priority</li> </ul>                                                                                                       |
| **Slack**                       | <ul> <li>Short line (day-to-day) of communication to Skyscrapers.</li> <li>`@help` as start of the support process for all Urgent requests and questions about the cooperation and processes (like requests needing special attention).</li> <li>General communication with the lead and other colleagues about Non-Urgent requests.</li> </ul> |
| **Dedicated 24/7 phone number** | For very urgent requests related to major production events requiring immediate attention, regardless of the time of day.                                                                                                                                                                                                                       |
| **Remote status meetings**      | Determine the status of the project and to see in which way we can help you achieve your goals.                                                                                                                                                                                                                                                 |

## Escalation process when production is down or hampered

1. If production is hampered and/or down, please use the automated number which is available in the main README file. Clearly state your name and describe the problem.
1. Our 24/7 on-call engineer is automatically alerted.
1. The engineer will acknowledge the problem and take the first steps towards a solution.
1. Closing:
   - The on-call engineer will notify you when everything is resolved
   - In case of major downtime caused by the hosting platform, we will produce a post-mortem document (within ~3 days)
