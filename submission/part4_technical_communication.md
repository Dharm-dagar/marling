# Part 4: Technical Communication

## Integrity Declaration

"I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words."

---

## Scenario Response

**Reviewer Question:** "Why did you choose this specific PR over the others? What made it comprehensible to you, and what challenges do you anticipate in implementing it?"

---

### My Response

I have selected PR#3877 (Web Read Only) as it is relatively close to my experience of web application and REST API development. I was familiar with Flask and Express and, as it became apparent, the middleware-like pattern applied here, where the HTTP requests are intercepted to implement access control before passing the handlers.

The scope of this PR is small and hence it is easy to comprehend. In contrast to other PRs which cut across multiple systems, including a PR that changes the core data classes, the web readonly PR is a simple, well-isolated PR. The web-based plugin is primarily connected with the database and library APIs, which means that I will be able to track the modifications without having to learn about the other internal parts such as the importer pipeline or the tagging logic.

This is also the problem that I have faced in practice. When I was operating my personal media server, I was forced to use nginx as a reverse proxy which has authentication to avoid illegal access. This would have been made easier by an in-built read-only mode that would have made the intention of such PR very clear to me.

I anticipate three key areas in terms of challenges. To start with, to properly fit with the request lifecycle of Flask, one needs to choose the least intrusive interception tool, which can be request hooks or middleware. Second, the testing can be difficult because of the necessity to develop web clients in various settings and work with the test fixtures of beets. Third, it is important to prevent backward compatibility by being careful with defaults and working with documentation.

These challenges would lead me to examine the configuration and testing of beets using the available beets plugins, especially examining test_web.py. I would also look at the request hook patterns of Flask and other projects and write focused tests initially in order to clearly define the way things should work before implementing the changes.
