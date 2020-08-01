## Opening (by Martin Fowler)

Tradable Quality: lower quality software => more features

External Quality (visible to customers) vs. Internal Quality

How to make the tradeoff?

See the graph at: https://martinfowler.com/bliki/DesignStaminaHypothesis.html

In the long run, software with better internal quality will be more easily modifiable and add more features to.

Low internal quality => hard to improve and evolve software.

With clean code, it takes a certain effort `x` to add a new feature. But with typical code, there is an additional effort `y` required on top of the effort `x`; `y` is the cost of Internal Quality.

For every problem, there is Essential Complexity and Accidental Complexity. It's kind of like Principal vs. Interest. The interest is the additional effort `y` required for new features.

Deliberate and Reckless: "We don't have time for architecture".
Deliberate and Prudent: "We must ship now and deal with consequences".
Inadvertent and Reckless: "What's layering?"

Reference graph at: https://martinfowler.com/bliki/DesignStaminaHypothesis.html

Tradeoffs only make sense below the Design Payoff line.

Many projects trading quality for speed are being Deliberate and Reckless.

Try explaining the economic reasons of why quality is important.


## Practices for an Agile Codebase (by Martin Fowler)

Responding to change over Following a plan (Part of Agile Manifesto)

How to get Internal Quality? (because Internal Quality leads to Responsive Features)

1. Self-testing code

- dev team builds tests along with each code. With single command, run all the tests; yes / no, green / red answer.
- Mechanism to spot mistakes when you make changes. Doesn't take much to find out what's wrong if you run it every few minutes.
- Brings confidence to change codebase. Lets you do all the shifting that otherwise would be impossible.

2. Refactoring

- Prerequisite: self-testing code.
- Make change if you spot something not quite right, eg. Confidently change any name on system and get told if it's correct or wrong.
- Keeps Internal Quality of software together.
- Self-testing code + Refactoring are necessary but not sufficient for Responsive Features.
- If you are working in a team, you need Continuous Integration.

3. CI

- Must push into mainline at least once a day.
- YOu get a lot more smaller integrations that are easier to deal with, rather than 1 big merge.
- If people make incompatible refactorings, they find out fast (contrast to big merges).
- Find out within a day. Communicate once find out.
- People doing feature branches are afraid of doing refactoring due to the breakages.

4. Yagni

- Beacuse the above lowers cost of change. So you don't have to design super far ahead (eg. 1 year, 2 years)
- But yagni is stupid if you don't have the above.
- Even right feature built right has a cost -> cost of carry (need to maintain and understand. Because it introduces complexity).

5. Continuous Delivery

- CI: gives you software working well in development environment
- CD: production. Heart of CD is deployment pipeline. It should bring more quality checks, eg. performance testing (expensive, cannot do all the time)
- Jez Humble: going through pipeline is like Greek Hero slaying demons until it gets to users.
- But that's not all. Monitoring system.
- Accelerate: a book on software development (https://www.amazon.com/Accelerate-Software-Performing-Technology-Organizations-ebook/dp/B07B9F83WM)


## A talk on collecting data for experiments

- Qualitative data: gives insights. Doesn't prove anything (may explain why something is happening). Eg. user interviews, usability testing.
- Quantitative data: tells you what is actually happening. Eg. A/B testing, eye tracking.
- Data: helps you solve problems and deduce hypothesis for experiments.
- No data -> Start with qualitative data as abaseline -> Come up with hypothesis -> Collect quantitaive data
- 2 key ingredients to successful experiment:
  - Sleecting the right idea
  - Fast feedback loop
- Netflix Experimentation Platform


## E2E ML pipelines

- Google's paper - ML: the high interest credit card of tech debt
- Find 1 business problem, collect minimal viable data, do feature engineering, train simple model, see if it helps.
- No data / data sucks: expose interface but don't use ML. Eg. chatbot using rules / regex. It is also a way to collect data
- Choosing between ifferent models: tracking and logging. A tool to help: MLFlow (can compare different models, track hyperparameters)
- Monitor service usage (requests, are people using it?)
- Monitor output (mainly to check for anomalies)
- Monitor inputs (identify training vs. serving skew. In other words, data in production that's not seen in training).
- LIME library: can help explain why model predicts something.
- Log down requests and response -> Data -> Get humans to evaluate if useful -> More training data
- https://thoughtworks.com/intelligent-empowerment
- Slides: https://tinyurl.com/ml-xconf
