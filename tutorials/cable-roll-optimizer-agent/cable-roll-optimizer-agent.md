---
author_name: Ilya Belozerov
author_profile: https://github.com/ilyabelozerovsap
keywords: tutorial
auto_validation: true
time: 30
tags: [software-product>joule studio, joule, joule work, tutorial>beginner, tutorial>license]
primary_tag: software-product>joule studio
parser: v2
---

# Optimize Cable Roll Selection for Work Orders with a Custom Joule Agent

## Prerequisites

- Access to **Joule Studio**
- Access to an **SAP S/4HANA Cloud** system with:
  - Production Orders (PP module) containing cable material requirements
  - Batch stock management across multiple storage locations (MM/WM module)
  - Stock Transfer Order creation enabled (MM module)

## You will learn

- How to identify a manufacturing workflow with a clear optimisation objective suitable for agent automation
- How to write an effective **intent statement** for a constraint-based optimisation agent
- How to review and validate the assets created, including the generated **Product Requirements Document (PRD)**
- How to deploy a production-ready Joule Agent to the SAP managed service

## Intro

>**IMPORTANT**
>
>**Welcome to the Agent lab**
>
>You are working with a pre-release version of the Joule Studio. This gives you an early look at our upcoming capabilities. Please keep the following in mind:
>
> - **Features are subject to change:** The UI, terminology, and functionality you see may differ from the final product.
> - **Educational use only:** This environment is designed for learning and experimentation, not for production use.
> - **Potential instability:** As a preview version, you may encounter occasional instability or unexpected behavior.

Cable is one of the most common sources of avoidable material waste in manufacturing. When a production work order specifies a required cable length, the operative must select a roll from available stock. The wrong choice - picking a roll that is significantly longer than needed - leaves an offcut that may be too short to reuse and is written off as scrap. Multiply that decision across hundreds of work orders per week, across multiple production lines and warehouse locations, and the cumulative waste becomes a measurable cost.

In this tutorial, you build a **Cable Roll Optimizer Agent** using Joule Studio. The agent reads work order cable requirements from SAP S/4HANA, checks available batch stock across all storage locations, applies a deterministic optimisation engine to identify the roll or combination of rolls that minimises offcut waste, and - where local stock is insufficient - recommends raising a stock transfer order (STO) to bring the right roll from another location. All recommendations require human approval before any SAP data is modified.

You'll follow the full Joule Studio development process - from a plain-language intent statement to a deployed Python agent integrated with SAP S/4HANA via dedicated MCP servers.

> **About the scenario:** The cable types, roll lengths, and location names used in this tutorial are illustrative. The agent pattern applies to any manufacturing context where a continuous material (cable, pipe, tubing, sheet, wire, fabric) is consumed by length or area from batch-managed stock rolls or coils held across multiple locations.

> **Why is this a strong automation candidate?** The optimisation logic is deterministic (minimise offcut = minimise roll_length minus required_length), the data sources are known (work order requirements + batch stock), the trigger is predictable (work order release), and the output is well-defined (roll selection + optional STO). Deterministic optimisation problems with known inputs and outputs are among the highest-value use cases for Joule Studio agents.

### Understand the Business Challenge

Before building the agent, it is important to understand the problem it solves. This step describes the three stock fulfillment scenarios the agent handles and the manual pain points it eliminates.

**Three cable fulfillment scenarios**

When a work order is released, the production planner or warehouse operative must fulfil the cable requirement from available stock. Three distinct scenarios arise:

| Scenario | Description | Challenge |
|----------|-------------|-----------|
| **Single-roll local fulfillment** | One roll at the work order's home location is sufficient for the required length. | Selecting the roll with the minimum offcut from potentially many available rolls of varying lengths requires manual comparison - and is often skipped in favour of the first available roll. |
| **Multi-roll local fulfillment** | No single roll covers the required length; two or more rolls from the local location must be combined. | Identifying the combination of rolls that covers the requirement with the least total offcut is a combinatorial problem that cannot be solved reliably by manual inspection. |
| **Cross-location fulfillment** | Local stock is insufficient in length or quantity. Suitable rolls exist at one or more other storage locations. | The planner has no consolidated view of roll availability across locations. Stock transfer orders are raised reactively and often for the wrong quantity, generating further waste at the source location. |

**Current pain points**

The manual handling of cable roll selection creates the following operational challenges:

- **No consolidated stock view** - Roll availability, batch numbers, and remaining lengths are visible per location but not aggregated across the network. Planners must check each location separately.
- **Suboptimal roll selection** - Without a systematic comparison, operatives select rolls based on proximity or habit rather than fit, generating avoidable offcut waste on every work order.
- **Untracked offcut accumulation** - Offcuts below a reusable threshold are not systematically identified at selection time, making waste invisible until a physical stock count.
- **Reactive stock transfers** - Cross-location replenishment is initiated only after local stock is exhausted, causing production delays and last-minute logistics pressure.
- **No audit link between selection and waste** - There is no record connecting the roll selection decision to the resulting offcut, making it impossible to measure or improve selection quality over time.

> The agent addresses all five pain points. It aggregates batch stock across all locations in a single query, applies a deterministic optimisation engine to every work order, surfaces the best roll selection with a quantified offcut estimate, and - where a stock transfer is needed - generates a ready-to-approve STO recommendation linked directly to the work order.

### Open Joule Studio and Define the Agent Intent

In this step, you select Agent as the solution type and describe what you want the agent to do. Joule Studio takes your intent and translates it into product requirements, a technical specification, and a complete implementation - no manual development required.

1. Open **Joule Work** and navigate to **Joule Studio** by selecting **Joule Studio** in the left navigation panel.

2. Select **Create** to open the **Create Agent** dialog. If this is not your first project in Joule Studio, select **+ New** in the **Solutions** panel to access the dialog.

![Open Joule Studio](bis1.1.png)

3. Enter the agent name:

    ```
    Cable Roll Optimizer
    ```

4. Enter the following intent statement in the description field:

    ```
    Create an agent that should optimize cable roll selection for manufacturing work orders to minimize offcut waste. It reads work order requirements from SAP S/4HANA - including cable material, required length, and specification - and checks available batch stock across all storage locations. Based on this data, the agent applies an optimization engine to recommend the single roll or combination of rolls that fulfills the work order with minimum waste. Where local stock is insufficient, the agent identifies available rolls at other locations and recommends raising a stock transfer order accordingly. All recommendations are presented to the planner for review and approval before any SAP data is modified. The agent supports configuration for acceptable waste thresholds, location priority rules, and batch selection criteria such as expiry date and certificate status.
    ```

5. Select **Quick Create** to skip clarifying questions and move directly to solution building. This adds *Fast Track* to the intent statement.

> Even with **Quick Create**, Joule will still ask you to confirm Business Goals & Success Criteria.

![Enter agent details](020-agent-details.png)

**Writing an effective intent statement**

A strong intent statement for an optimisation agent covers four dimensions: the precise mathematical objective (minimise offcut), the data it works with (work order requirements + batch stock), the fulfillment scenarios it must handle (single-roll, multi-roll, cross-location), and the guardrails it must respect (approval before any write). Explicitly naming the optimisation objective - rather than describing it only as a recommendation - gives Joule Studio the context it needs to generate a deterministic engine rather than an LLM-based selection. If the generated interpretation doesn't match your expectations, refine the statement here before continuing.

6. Select **Create** to proceed.

7. Confirm the Business Goals.

Even when **Quick Create** is selected, Joule will collect the mandatory Business Goals & Success Criteria before completing the intent analysis. Answer any additional clarifying questions to the best of your knowledge - these inputs help Joule define the success metrics and tailor the generated PRD.

![Confirm business goals](025-business-goals.png)

### Review the Intent

1. After you submit the intent statement, Joule Studio generates an **intent.md** and presents a summary.

2. Under the **Intent Summary** tab, you may review the complete summary. This shows how Joule Studio interpreted your input.

![Intent Summary](035-intent-summary.png)

3. Verify that this matches your intent. If the summary is missing an element or describes a different scope, you can refine the intent statement and re-submit.

4. You can also find the **intent.md** file under **File tree** panel.

5. Scroll down to the **Fit Gap Analysis** and see what Joule has identified.

6. Then you can scroll down to the **Recommended Solution**. It describes a Python-based agent with a dedicated deterministic optimisation engine (cutting stock algorithm) separated from the LLM narration layer, and what the agent should be able to do.

![Exploring the Intent File](036-intent-recommended-solution.png)

### Product Requirements Document

In the **Requirements** phase, Joule Studio generates a **Product Requirements Document (PRD)**. This is an important document that you are expected to review carefully. This document defines the agent's operational boundaries and becomes the source of truth for all subsequent phases.

1. Wait until the requirements have been generated.

2. Choose **Requirements** under **Solution Progress** panel. This opens the low-code view of the PRD.

![PRD](060-solution-prd-creation-in-progress.png)

3. To access the raw Markdown file, move to the **File tree** panel - this is the code view of the same document.

The Business Context section maps your intent to the actual business process - it identifies personas (e.g., production planner, warehouse operative), pain points, success metrics, and guardrails. Review this section carefully: it defines the operational boundaries the agent will respect throughout all subsequent phases.

**Product Purpose and Value Proposition**

At the top of the document, you will find the elevator pitch, the business need and expected value described. The product objectives are also listed.

**Must-Have Requirements**

This section provides ranked requirements, with their user stories and acceptance criteria.

**Automation and Agent Behavior**

This section is critical - it defines exactly what the agent does autonomously, what requires a human decision, and what guardrails are in place.

The agent operates at a **hybrid automation level**:

- *Autonomous actions (no human required)*, such as retrieving work order requirements and running the optimisation engine
- *Human approval required*, such as raising a stock transfer order in SAP S/4HANA

**Engine vs. LLM Boundaries**

The optimisation calculation - selecting rolls to minimise offcut - is performed entirely by a **deterministic engine** using a constrained cutting stock algorithm. The LLM is used exclusively for narrating the recommendation in natural language and explaining the trade-offs when multiple options exist. Any numeric figures (roll lengths, offcut estimates, transfer quantities) presented by the LLM are sourced directly from the engine output and are not independently generated.

**Guardrails**

Four operational guardrails are built into the agent:

1. **No STO is raised** in SAP S/4HANA without a recorded planner approval signal.
2. **Batch certificate and expiry validation** - rolls that fail the work order's specification requirements are excluded from the optimisation candidate set before any recommendation is made.
3. **Waste threshold escalation** - if the best available roll selection exceeds the configured offcut waste threshold (e.g. more than 15% of the roll length), the recommendation is flagged as requiring planner review rather than being presented as a standard approval.
4. **Stock data freshness check** - the agent verifies that batch stock data is current before running the optimisation; stale data triggers a re-query and a notification if the refresh fails.

**Solution Architecture**

In the solution architecture section you will see the proposed components and the integration points. The main components are: the agent itself, dedicated MCP servers to access the SAP S/4HANA backend, and SAP AI Core to enable LLM access for the agent.

![Architecture](070-solution-architecture.png)

### Inspect the Generated Specification

In the **Specification** phase, Joule Studio translates the PRD into a generated file tree: a complete, structured, and reviewable set of assets and configuration files. No code has been written yet; this is the blueprint.

1. Wait for the specification generation to complete. Review the summary of what has been created so far.

![Specification Summary](075-inspect-specification.png)

The entire specification is transparent and reviewable **before any deployment is triggered**.

> **Architecture note:** The Cable Roll Optimizer Agent uses **dedicated MCP servers** for different concerns: a read-only Production Order MCP server for work order data (PP module), and a Batch Stock MCP server (MM/WM module) that is read-only for stock queries and write-enabled exclusively for STO creation upon planner approval. Keeping the servers separate makes the integration boundaries explicit and independently auditable.

**Example of the `assets/` folder**

```
assets/
├── cable-roll-optimizer-agent/        # Core agent:
│                                      # - Cutting stock optimisation engine
│                                      #   (minimise offcut, single/multi-roll,
│                                      #    cross-location scenarios)
│                                      # - Batch certificate and expiry validator
│                                      # - Waste threshold evaluator
│                                      # - LLM narration prompts for recommendations
│                                      # - Approval handling and STO trigger logic
│                                      # - Audit logger
│
├── sap-s4-productionorder-mcp-server/ # MCP server: Production Order API (PP)
│                                      # Exposes: work order number, cable material,
│                                      # required length, specification, production
│                                      # location, release status
│                                      # Access: read-only
│
└── sap-s4-batchstock-mcp-server/      # MCP server: Batch Stock & STO API (MM/WM)
                                       # Exposes: batch inventory by location,
                                       # roll lengths, certificate status, expiry;
                                       # Stock Transfer Order creation (write,
                                       # invoked only on planner approval)
                                       # Access: read for stock queries;
                                       #         write for STO creation only
```

> **Why a deterministic engine rather than an LLM for optimisation?**
> Roll selection is a well-defined mathematical problem (a variant of the one-dimensional cutting stock problem). A deterministic engine guarantees consistent, reproducible, and auditable results - the same work order and stock inputs always produce the same recommendation. An LLM cannot guarantee this. Joule Studio generates the deterministic engine from the PRD specification; the LLM is invoked only to narrate the engine's output.

### Generate the Solution

1. Once the specification is ready, Joule Studio will proceed with the solution generation.

In the **Solution Artifacts** phase, Joule Studio executes the specification and generates the complete, runnable solution: the optimisation engine, the MCP server integrations, the approval flow, and the STO creation logic - with no manual development required. Automatic tests will be created and executed.

2. Wait for the solution generation to complete before proceeding. You can find the agent under **Solution Artifacts**.

![Agent](080-solution-fully-built.png)

During this phase, Joule Studio:

- Scaffolds the full Python agent with the cutting stock optimisation engine, the batch validator, and the waste threshold evaluator from the `assets/` tree
- Wires up the Production Order and Batch Stock STO API bindings for the dedicated MCP Servers
- Configures the **SAP Generative AI Hub** connection for LLM-based recommendation narration
- Implements the planner approval gate and the STO trigger logic, ensuring the write operation on the MCP Server is only reachable via the approval code path
- Applies the four operational guardrails, including the batch certificate validator and the waste threshold escalation logic
- Instruments all agent actions - work order reads, stock queries, optimisation results, recommendations, approval events, and STO creation - with audit logging

**Reviewing the Agent Definition**

1. Select the agent under **Solution Artifacts** to open its definition. In the low-code view you can see:

- The LLM configuration (e.g., sap/anthropic--claude-4.5-sonnet)
- Agent Configurations: circuit-breaker threshold, thread TTL, summarization trigger
- The MCP Servers that allow the agent to read work orders and batch stock from SAP S/4HANA, and to raise stock transfer orders upon planner approval

![Agent configuration](100-agent-configuration-overview.png)

4. In the **Evaluation** section you can generate evaluation scenarios for this agent based on your intent.

![Agent evaluation](100-evaluation-section.png)

5. To see the generated Python code, move to the pro-code (File tree) view.

![Code](110-code-overview.png)

**Reviewing the MCP Servers**

Select **MCP Servers** in the agent definition to see the servers that were configured automatically. Each one gives the agent access to a specific system:

- The **Production Order MCP server** connects to the SAP S/4HANA Production Order API (PP module), giving the agent read access to cable material numbers, required lengths, specifications, and production locations.
- The **Batch Stock MCP server** connects to the SAP S/4HANA Batch Stock and STO API (MM/WM module), giving the agent read access to roll inventory across all configured locations and write access - scoped exclusively to STO creation - upon planner approval.

![MCP](120-mcp-review.png)

No configuration is required: Joule Studio generated and wired up all servers from the specification.

Once Joule confirms the solution build is complete, proceed to **Testing Overview** to see the automated test results.

> If Joule Studio reports a warning about the Batch Stock MCP server scope during generation, check that the storage location codes in `solution.yaml` match the location identifiers in your SAP S/4HANA plant structure. A mismatch here will cause the stock query to return empty results in testing.

### Validate the Agent

The **Testing** phase is triggered automatically by Joule once the solution build is complete. It runs an automated validation suite derived from the specification - you do not need to initiate it manually. Wait for Joule to confirm that testing has finished before proceeding.

1. Once testing is complete, open **Overview** under **Testing** panel. Review the results and compare the test categories to your specification to verify all requirements have been validated.

2. Verify that all tests pass before proceeding to deployment.

![Verify Solution](090-tests-overview.png)

> If any test fails, Joule will usually try to fix it automatically. If it gets stuck, you might converse with Joule to help fix the problem.

3. Select the agent under **Preview** panel.

4. In the chat field, enter a prompt that does not require live tool calls. For example: *What cable requirements can you help me optimise?* The agent will respond with a description of its capabilities without needing to connect to SAP S/4HANA.

![Testing start](130-test-agent-locally.png)

![Test results](130-local-test-results.png)

You can do some further testing before deployment.

> **If the single-roll optimisation tests pass but multi-roll tests fail**, this typically indicates a constraint configuration issue in the cutting stock engine rather than an API problem. Review the `max_rolls_per_order` parameter in `solution.yaml` and the combination logic in `cable-roll-optimizer-agent/engine/cutting_stock.py`.

> **If the STO creation test fails**, check that the Batch Stock MCP server's write credential has the correct SAP S/4HANA authorisation object for stock transfer order creation (`M_MSEG_BWA` with movement type 351 or equivalent for your system configuration).

### Deploy the Agent to Development Landscape

With all tests passed and the validation score confirmed, the **Cable Roll Optimizer Agent** is ready for deployment.

1. Choose **Deploy**. Then confirm with **Deploy** again.

![Deploy](140-deploy-solution.png)

Joule Studio packages the agent and both MCP servers and deploys them to the **SAP managed service** on BAIP. Once deployment completes, the `deploy_result.json` file in the `specification/` folder is populated with the live deployment details (endpoint URL, runtime ID, deployment timestamp).

Your agent is now operational and available to production planners.

> If you are working in a team or want to version-control the generated code, Joule Studio supports GitHub sync. You can find this option under **Actions** in the top bar.

> For governance reasons, deployment to production is not done from within Joule Studio.

**How the agent works in production**

Once deployed, agents are automatically discoverable through Joule. When users ask questions that match the agent's area of responsibility, Joule routes the request to the agent. Users don't need to know that a custom agent exists or how to find it.

**Monitoring the Agent**

Once deployed, switch to the **Manage** tab in Joule Studio to access governance and monitoring. Here you can view runtime status, active deployments, usage metrics, and audit logs.

### Summary

You have completed the end-to-end design and deployment of a Cable Roll Optimizer Agent using Joule Studio. In doing so, you have:

- Identified the three cable fulfillment scenarios (single-roll local, multi-roll local, cross-location) and the five manual pain points the agent addresses
- Written an effective intent statement that explicitly describes the optimisation objective, the fulfillment scenarios, the approval gate, and the configurability requirements - giving Joule Studio the context to generate a deterministic engine rather than an LLM-based selection
- Reviewed and validated the assets created, including the generated PRD defining product objectives, a hybrid automation level, explicit engine vs. LLM boundaries, and four operational guardrails
- Inspected a generated file tree with dedicated MCP servers - a read-only Production Order server and a read/write Batch Stock server scoped to approve-gated STO creation
- Reviewed automated test coverage across single-roll, multi-roll, and cross-location optimisation scenarios, certificate validation, threshold escalation, approval gate enforcement, and STO creation
- Deployed the agent to the SAP managed service

The agent replaces an ad hoc, intuition-based, per-location selection process with a **systematic, data-driven, auditable optimisation** - reducing offcut waste on every work order, surfacing cross-location stock proactively, and creating the audit trail needed to measure and continuously improve material efficiency.
