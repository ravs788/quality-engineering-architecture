Context
I was involved in designing the test architecture for an audit domain project. The solution consisted of two products
- the main application was a desktop (wpf + angular) based application which was used by auditors to track their audit processes. Each audit was an individual entity called engagement and could be shared by multiple auditors who could work in online as well as offline mode. the data is synced regulalrly even in offline mode at specific points of time. If multiple people are working on the same entity and they make contrasting changes, there is an option provided to resolve this by choosing which of this changes to select.
- the 2nd application is a configuration application - it is a web application - based on certain options choosen while creating engagement - the internal pages of the applciation would be created.

- release process was a 4-6 month one - 1 month sprints involving development and testing of specific features
- 2-3 months of testing by different set of people - testers at our end, business, UAT etc.
- final release to production

Decision
Major points here - these were already done
- start with a large manual testing suite developed over major releases.
- catgeorize these into different priorities (P1-P4) and group them - add labels
- start looking to automate the P1s and if not existing for any categories add P2s for them.
- run them manually on a daily basis and update the details in Azure Devops manually.
- have a small performance test suite which covered some basic functionality and it was run and updated once a sprint and during detailed testing phases in each cycle.

decisions made
- analyze manual test suite to determine what factors could be covered at lower layers - unit tests, testing apis
- started adding more unit, component and integration tests
- started removing these from UI layer - keeping 1-2 for redundancy
- add more tags and info on the manual tests to determine
    - any priority changes
    - what is redundant
    - what can be covered at lower layers - unit, api etc.
    - what should be automated and what shouldn't be
- revisit UI automation - 
    - a lot of time was spent in running the tests 
    - tests were run on physical VMs
    - workflows were completely UI based
    - parallelization wasn't possible for desktop based application
    - application updates if any were done manually on each VM
- changes suggested
    - use Kubernetes to orchestrate VM creation at execution time
    - use docker images to repro the environment in the Kuberenetes pods
    - use scripts for orchestrarting application upgrades on Kuberenetes pods
    - make use of APIs to create test data before trigerring execution only for the relevant test verifications.
    - revisit tests every release and discard tests too and not just add new ones
    - entire test suite run everyday
    - a small smoke test suite added to CI pipeline to indicate build health
- performance testing 
    - was using a soon to be deprecated MS tool
    - suport for the tool was not readily available
changes suggested
    - start evaluating more supported tools - Jmeter vs K6
    - Jmeter was chosen due to its wide support and flexibility
    - migration plan finalized
    - revisited the test suite - adding and removing tests as needed
    - revisited the SLAs for all scenarios

Trade-Offs

Outcome