# Feeding GenAI: Data Sources, Serialization and other tricks

This is the high-level outline of a presentation. I'd like to talk about the lessons learned during implementing
a retail market analytics platform.

## Semantic Layer

The application needs to aggregate business metrics across several dimensions. The traditional method is
to write custom SQL queries that contain the expressions associated with each metric.

The solution is brittle, and metric definitions can diverge between different applications.

Databricks Metric Views provide a way to define business metrics in a unified way.

Demo:
- Show original SQL
- Show metric view definition YAML files
- Show derived SQL with the "MEASURE" aggregate function

Highlight that non-transitive expressions like average and ratios also work seamlessly.

## Serialization

Even in the multi-modal LLM world, we need to supply data as text. How do we serialize a dozen of data sets?
- The classic solution is JSON
- Introduce the TOON format that is 70% shorter and more readable.

Demo
- Show the original JSON file. Highlight the repetition of the attributes.
- Show the TOON formatted file. Highlight the conciseness
- Show the size difference between the files

Mention that the toons python library just a wrapper around a performant RUST code.
It contained an error when serializing certain unicode characters. I fixed it with claude without knowing RUST.

## Handling "Conclusions and Insights"

The generated report had a "Conclusions and Insights" section generated at the end.
The customer asked to move it forward, just after the overview.

Possible solutions:
* Simply change the system prompt, and ask to generate it at the beginning. 
  Drawback: The section cannot benefit from the temporary findings during reporting. 
  The section will become less insightful.
* Split the report generation into two: Generate the report first, then send the generated report through another LLM
  that creates and ingests the insights. 
  Drawback: The solution is slower and more expensive.
* Text processing: parse the generated markdown file and move the section forward. 
  Drawback: generated text is unpredictable and text processing can be brittle.

Applied solution:

Used structured output: asked to generate the report in JSON format with the attributes 
  {title, overview, detailed_report, conclusions_and_insights}. 
  Translated the JSON file to text by changing the order.