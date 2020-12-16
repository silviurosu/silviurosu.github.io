1) what questions should a developer ask himself
    * is the requirement clear? Do I need more clarification before starting to dig into the code?
    * how can I test this requirement? Let's write first an test that guarantees a working solution
    * Is this code a piece of a greater flow? Do I need an integration test to cover the whole flow?
    * How can I write the code easy to read? Does it require additional documentation? Does a different developer can understand what was my logic?
    * Is there a pattern that is reused in the code? Can it be extracted as an utility?
    * How fast this code runs? Can I have a benchmark on top?
    * Does this code requires metrics? It is a functionality that needs to have alerts when something does not work as expected

2) custom metrics with telemetry
3)