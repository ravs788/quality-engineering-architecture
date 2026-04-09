# Refined Test Architecture for Audit Domain Project

## 1. Context

The audit domain solution consisted of two connected products:

- **Desktop application (WPF + Angular):** used by auditors to manage audit work for an engagement. Multiple auditors could work on the same engagement in both online and offline modes. Data synchronization happened at defined intervals, and conflicting changes required explicit resolution.
- **Configuration application (Web):** used to define engagement options. Based on the selected options, the internal pages and behavior of the engagement experience were generated.

The release model was long-running and validation-heavy:

- **Development phase:** 4-6 month release cycle with 1-month sprints
- **Detailed test phase:** 2-3 months of testing by QA, business stakeholders, and UAT teams
- **Release phase:** production deployment after final validation

This domain introduced a set of risks that the test architecture had to address directly:

- correctness of offline and online synchronization
- concurrent updates by multiple auditors on the same engagement
- conflict detection and conflict resolution behavior
- correctness of configuration-driven application behavior
- regression risk across a long release cycle with multiple validation stages

## 2. Problem Statement

The original test approach relied heavily on a large manual regression suite accumulated over multiple releases. That approach provided coverage, but it created scaling and feedback problems:

- critical tests existed, but execution cost was high
- many workflows were verified only through the UI layer
- the desktop automation estate was slow to execute and difficult to maintain
- execution depended on physical VMs and manual application updates
- manual daily execution and Azure DevOps result updates consumed significant effort
- performance coverage existed, but it depended on a Microsoft tool that was approaching deprecation and had limited support

The central architecture problem was not lack of tests. It was that too much validation lived in the most expensive layer, which slowed feedback and made the regression portfolio harder to sustain across releases.

## 3. Decision

The test architecture was refined around a layered validation model, using the manual suite as the starting point rather than replacing it all at once.

### 3.1 Baseline and Prioritization

The existing manual suite remained the initial source of coverage. Tests were categorized into priorities such as P1-P4 and grouped with labels so the suite could be managed more deliberately.

The suite was then reviewed to answer four questions for each scenario:

- Is the priority still correct?
- Is the test redundant with another test?
- Can the behavior be validated at a lower layer such as unit, component, integration, or API?
- Should the scenario remain manual, or should it be automated?

### 3.2 Layered Test Strategy

The main architectural decision was to move as much verification as possible out of the UI layer and into lower, faster, and more targeted layers.

| Concern | Preferred Layer | Rationale |
|---|---|---|
| business rules and conflict logic | unit tests | fast validation of deterministic behavior |
| component behavior and page-level logic | component tests | validates isolated functional behavior without full end-to-end cost |
| service and integration contracts | integration and API tests | validates system interaction and data flow |
| end-to-end engagement workflows | UI automation | retained only for a focused set of critical paths |
| exploratory and rare edge scenarios | manual testing | preserves human evaluation where automation is low value |

This led to three concrete actions:

- increase unit, component, integration, and API coverage
- remove redundant UI scenarios once equivalent lower-layer coverage existed
- keep a very small number of UI tests for critical-path redundancy

### 3.3 Manual and Automated Portfolio Management

The manual suite was treated as a living asset, not a static backlog. Additional tagging and metadata were introduced so the team could continuously decide:

- what still mattered
- what no longer justified execution cost
- what had already been covered elsewhere
- what should move into automation next

The operating principle was that tests should be removed when they stopped adding value, not only added when new features appeared.

### 3.4 Execution Model for UI Automation

UI automation remained necessary for a desktop product with complex cross-screen workflows, but the way those tests were executed had to change.

The previous execution model created four recurring problems:

- runs were slow
- execution depended heavily on long-lived physical or manually maintained VMs
- desktop-driven scenarios were difficult to parallelize effectively
- upgrading the application and preparing the environment required repeated manual effort

The refined direction was to make execution more reproducible and less dependent on manually prepared environments, without pretending that desktop UI execution could be parallelized freely on a single VM. The proposal combined standardized Docker images with scripted preparation of VM-based execution targets:

- use Docker images for supporting services and tooling so environments could be provisioned more consistently on the VM-based test infrastructure
- release upgraded versions of the images so application and supporting infrastructure changes could be deployed quickly onto the VMs instead of being updated manually on each machine
- script application installation and environment preparation steps instead of updating each execution target by hand
- create prerequisite data through APIs where possible so UI automation spent time on workflow verification rather than setup
- accept that desktop application flows could not be parallelized effectively on a single VM, and scale execution only by adding more prepared machines when needed

For a desktop-heavy product, the core execution model still depended on Windows-capable VMs for the UI automation itself. The improvement was not full infrastructure abstraction. It was the reduction of environment drift and manual setup effort through repeatable images, scripted upgrades, and cleaner separation between environment preparation and test execution.

### 3.5 Test Cadence

The execution strategy was split by feedback speed:

- **CI pipeline:** a small smoke suite to indicate build health quickly
- **Daily execution:** broader regression coverage for the active test estate
- **Release-cycle validation:** deeper manual, business, and UAT validation during the longer test phase

This preserved rapid build feedback without forcing the full end-to-end portfolio into every pipeline run.

### 3.6 Performance Test Architecture

The performance test suite remained intentionally small and focused on core business scenarios, but its technical foundation was changed.

The existing Microsoft performance tool was becoming a liability because of impending deprecation and limited support. The team therefore evaluated better-supported alternatives, including JMeter and k6.

JMeter was selected because it offered:

- broader support and community knowledge
- sufficient flexibility for the required scenarios
- a practical migration path from the existing suite

The performance workstream then focused on:

- defining a migration plan
- revisiting the existing performance scenarios
- adding or removing scenarios based on current risk
- revisiting SLA expectations for the covered workflows

### 3.7 Alternatives Considered

Several alternatives were considered implicitly during the evolution of the architecture, even when they were not all formalized as separate design documents.

- **Keep broad UI regression as the primary safety net:** rejected because execution cost, maintenance effort, and feedback latency were already too high.
- **Expand automation without pruning the manual suite:** rejected because it would grow total validation cost without improving portfolio quality.
- **Rely mainly on long-lived physical VM infrastructure:** rejected because manual updates and environment drift made the execution model hard to scale.
- **Adopt another performance tool such as k6:** considered, but JMeter was selected because it had broader support in the team context and a more practical migration path from the existing suite.

## 4. Trade-Offs

The refined architecture improved sustainability and feedback speed, but it was not free of trade-offs.

- Moving coverage to lower layers required upfront analysis and test redesign.
- Retaining a small UI redundancy set meant accepting some duplicate coverage in exchange for critical-path confidence.
- Desktop automation could not be optimized in the same way as stateless web-only tests, so environment orchestration still required deliberate engineering.
- Treating the manual suite as a curated portfolio required ongoing governance instead of one-time cleanup.
- Migrating performance tooling introduced short-term transition cost in exchange for better long-term maintainability.

## 5. Outcome

The resulting test architecture was more intentional than the original manual-heavy model.

- The manual suite became the source for prioritization and architecture decisions rather than the default execution layer for every scenario.
- More validation was pushed into unit, component, integration, and API layers, reducing unnecessary UI dependence.
- UI automation was repositioned as a focused critical-path safety net rather than the primary mechanism for broad regression coverage.
- Daily execution and CI smoke coverage improved feedback structure across the release cycle.
- Performance testing moved toward a supportable toolchain with refreshed scenarios and revised SLA alignment.

### 5.1 Measurable Impact

The architecture changes produced visible operational improvements in the regression process.

- daily UI regression execution was reduced from `8 hours` to `4.5 hours`
- approximately `500` high-value scenarios were moved from UI or manual validation into lower-layer automation
- environment preparation effort dropped from roughly `2 hours` each day to about `15 minutes`

These changes mattered not only because they reduced execution cost, but because they improved the speed and sustainability of the feedback loop. The team was able to spend less time preparing and running expensive UI validation and more time using lower-layer automation for faster, more targeted confidence.

## 6. Lessons Learned

The main lesson from this work was that test architecture improvement came less from adding more automation and more from redistributing validation to the right layers. The manual estate had to be treated as a managed portfolio, not as a permanent regression backlog. In retrospect, the most valuable decisions were the ones that reduced UI dependence, made environment setup more reproducible, and forced regular removal of low-value tests rather than allowing the suite to grow unchecked.

## 7. Architecture Summary

The key architectural shift was from "large manual regression with expensive UI automation support" to "risk-based layered validation with a curated manual suite and targeted desktop automation."

That shift was appropriate for this domain because the main risks were not simply page-level regressions. They were synchronization correctness, conflict handling, multi-user collaboration, and sustained regression control over a long release cycle. The refined architecture improved the team's ability to validate those risks using the cheapest reliable layer first, while still retaining selective end-to-end coverage where it mattered most.
