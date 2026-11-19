# Demo: AI Agents That Understand Business Context

**Codex Desktop and Codex in VS Code + Business Knowledge Skill + MCP + dbt MetricFlow**

These walkthroughs illustrate how an AI assistant can combine **documented business meaning** with **queried metrics**. 


## 1. A user can use a product without being a paying customer

**Question:** "Do these organizations count as current customers?"

**Business Definition:** A **Customer** is an organization at the **Ultimate Parent level** with **non-zero ARR from active contracts**

**Why it is difficult:** a school or district can have licenses and usage while the commercial agreement sits with another organization. Product usage and active ARR are not interchangeable.
In addition, only a district is counted as a customer, not individual schools.

**How the agent answers:**

1. Uses the Business Knowledge Skill to identify the customer definition.
2. Resolves account identities and ultimate-parent reporting relationships.
3. Retrieves relevant current-period ARR using MCP and MetricFlow.
4. Explains which ultimate-parent entity is counted; avoids double-counting schools and districts.


`A user asks if some accounts are customers and Codex using customer definition from the ontolgy :`

![A user asks if some accounts are customers and Codex using customer definition from the ontolgy](./img/Customer%20Definition%201.png)

`Based on the customer definition in the ontology, metric values using dbt MetricFlow and account dimensional attributes to get ultimate parent account, Codex defines customers and explains the decision:`

![Based on the customer definition in the ontology, metric values using dbt MetricFlow and account dimensional attributes to get ultimate parent account, Codex defines customers and explains the decision.](./img/Customer%20Definition%202.png)


**What this demonstrates:** ontology-based interpretation changes the *meaning* of the answer, not simply its wording.



## 2. Usage without ARR: operational vs. commercial context

**Question:** "How can a district have product usage and zero ARR?"

The business ontology distinguishes an organization that **uses** a product from the entity that **funds** access. In a state-funded arrangement, the benefiting district may have active usage without qualifying as an individual paying customer. It is important not to infer a particular funding contract solely from a state eligibility flag.

`A user asks a clarification question regarding an account with usage:`
![A user asks a clarification question regarding an account with usage.](./img/Customer%20Definition%203.png)

**What this demonstrates:** the agent uses a documented business rule to explain an apparent contradiction while preserving the limits of the underlying data.



## 3. An account profile backed by multiple business perspectives

**Question:** "What do we know about this district in our internal data?"

The response may combine available account identity, commercial activity, active contract ARR, license provisioning, usage, support, and training. The account lookup provides structured organizational context; metrics are requested separately through the governed semantic query interface.

![Account detail information](./img/Account%20Details%201.png)


**What this demonstrates:** a business-oriented answer can bring together dimensions and metrics without pretending that the business has one perfectly aligned organizational hierarchy.



## 4. Generate an account-specific knowledge graph

**Question:** "Draw a knowledge graph of this district and its current metrics."

The agent uses the Markdown ontology's relationship vocabulary to connect the organization to operational and commercial representations, usage, opportunities, licenses, and queried measures.

**Important distinction:** this is an **agent-generated knowledge-graph visualization**, not a claim that the repository contains a deployed RDF/OWL graph store or graph-database ingestion pipeline. The business relationships come from documentation; the instance values come from accessible data tools.

![Account detail information as a knnowledge graph](./img/Account%20Details%202.png)

## 5. Similar names must not be silently merged

**Question:** "Tell me about XYZ school district."

A name search may return multiple organizations with exactly the same name like schools or districts in different states or **trial or evaluation records**. `get_account` returns candidate matches when identity is ambiguous, allowing the agent to request state/district clarification instead of choosing an arbitrary record.

`More than one account with the same name found in different states. Codex asks which one should be used:`
![More than one account with the same name found in different states. Codex asks which one should be used](./img/Ambiguity%201.png)

`Evaluation and trial account found with the same name. Codex asks if the exact account should be used, not evaluation or trial acounts:`

![Evaluation and trial account found with the same name. Codex asks if the exact account should be used, not evaluation or trial acounts](./img/Ambiguity%202.png)

`Confirmation:`

![Confirmation](./img/Ambiguity%203.png)

## 6. Explore ARR with tables and trends

**Question:** "Show Preliminary ARR by fiscal month, comparing two fiscal years"

The Skill defines ARR types and fiscal-year terminology; MetricFlow supplies data through semantic metrics. The agent can present a monthly table, comparison, or chart without redefining ARR inside a prompt.

This example illustrates why the distinction between **Preliminary, True, and Backdated ARR** belongs in documented business knowledge, while their numeric values come from the semantic metrics interface.

`Table Output`

![Table Output](./img/Metrics%201.png)


`Chart Output (Preliminary ARR is used by default if a user does not specify ARR type)`

![Chart Output](./img/Metrics%202.png)


## The key idea

| Layer | Question it answers |
|---|---|
| Ontology and glossary | What is a customer? How do accounts, contracts, licenses, and usage relate? |
| Skill and agent instructions | Which definitions and tools should the agent use? What should it refuse to assume? |
| dbt Semantic Layer / MetricFlow | Which metric is requested, by which dimensions and period, and how is its SQL constructed? |
| MCP server | What controlled operations can the agent perform? How are requests validated? |
| Codex | How should retrieved facts and metric results be explained and visualized? |

## Reproducing the approach

Start with the [main project overview](../../README.md), then review the [Skill](../../plugins/demoed-business-context/skills/demoed-business-context/SKILL.md), [agent instructions](../../AGENTS.md), and [MCP server README](../../workbench/metricflow_mcp/README.md).

The original data and full dbt project are not distributed here. Reproducing this design independently requires your own authorized warehouse, dbt semantic models, account data mapping, and an MCP configuration pointing to them.
