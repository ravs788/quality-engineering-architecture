Context
1. I started on this new test automation project managing 3 teams working across different areas of the application. 
2. The application development was done in C# and it consisted of a Web application, a desktop application and a set of APIs. 3. These were separate entities serving separate tasks and were not interrelated. 
4. There was manual test cases and manual testing done and no test automation. 
5. We were supposed to use the C# stack along with whatever tools required for the tasks. 
6. We decided to choose
    a. Selenium with C# for web automation,
    b. CodedUI for Desktop application
    c. RestSharp for API test automation.

Decsisions Taken
1. Created a comprehensive generic framework.
2. It had support for all 3 - desktop, api and web applications.
3. It had roughly similar structure as the solution at /Users/ravishankar/Desktop/Learning/Testing/complete-test-automation-fmw.
4. We made use of tfs for both storing the test automation as well as for CI integration.
5. We defined the core architecture and reviewed with the product development teams and other key stakeholders.
6. We build plans and defined teams to make progress across all 3 areas.
7. We defined a core team that focused on frameowrk activities.
8. We defined other teams for specific applications which focused on creating automation scripts for existing manual regression test cases and they worked closely with the application development and testing teams.
9. Our focus from day 1 was to get all test created running in CI as soon as possible.
10. Each test artifacrt was reviewed by our team as well as by the development and test teams.
11. We defined a common reporting tool - Extent reports for test reporting and a common elastic based logging framework already used by the teams for logging purpose.
12. Our main focus was on setting up a strong common farmework that could be extended across other applications and types if needed even if it required a bit more effort to spend initially in getting this ready.
13. Our focus was on getting parallelized execution in as quickly as possible - especially for the API and Web layers which would reduce execution time of the suites in the long run.

Outcome and Impact
1. We build test suites of 300 API tests, 200 desktop application tests and 250 Browser UI tests in a period of 4 months.
2. In addition to the above, we also helped the application teams move about 30% of their tests from UI layers to lower layers - unit, component, API.
3. There were some challenges initially as we required some time to get in sync with the expectations of the application teams.
4. We started running tests in CI as soon as possible and we were able to run the suites in
    a. API - 15 mins - There were hardly any API tests before this.
    b. Desktop - 1 hr - Used to take 2 person days of effort, by parallelizing things, reprioritizing tests, chossing the most relevant ones and using multiple VMs we brought this down tremendously.
    c. Browser - 30 mins - We chose a thin layer to start and built a strong, relevant, high-priority suite which used to take 3 person days previously to execute. We also supported multi-browser testing for specific scenarios.
5. We also transitioned the development and maintenance activities for the project when we moved out, training the team and building strong documentation that would aid them well into the future.
6. The long term affect was us gaining foothold in the client's business, starting 5-6 aditional projects across different tech stacks requiring building of test automation across multiple layers as needed.