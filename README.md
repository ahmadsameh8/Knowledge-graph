## Competition Law Cases – Data Preparation & Knowledge Graph

This project prepares competition law case data for analysis and for building a Neo4j-based knowledge graph.  
The core logic currently lives in the Jupyter notebook `Cleaning.ipynb`, with Cypher scripts and visual material in the root folder.  

### Data Source

The underlying case data are **publicly available** from the European Commission competition cases search portal:  
[`https://competition-cases.ec.europa.eu/search`](https://competition-cases.ec.europa.eu/search)

### Project Structure

- **`Cleaning.ipynb`**: End‑to‑end data cleaning and transformation pipeline.
- **`analysis.ipynb`**: Downstream analysis / exploration of the cleaned data (e.g., for charts or KG design).
- **`case-data-AT.json`, `case-data-M.json`**: Raw JSON data downloded from European Commission competition cases website.
- **`cases.json`**: cleaned JSON (produced by `Cleaning.ipynb`).
- **`cases.csv`**: Tabular export of `cases.json` suitable for analysis or import into neo4j.
- **`build_queries.cypher`, `visualize_query.cypher`**: Cypher scripts to build and explore the Neo4j knowledge graph.

### Data Pipeline in `Cleaning.ipynb`

The notebook performs the following main steps:

- **Extract & consolidate cases**
  - Function `extract_cases(source_file, target_file)` loads raw JSON (`case-data-M.json`, `case-data-AT.json`).
  - Produces a compact structure with key metadata:
    - `caseInstrument`, `caseNumber`, `caseTitle`
    - `caseSectorsCode`, `caseSectorLabel`
    - `caseCompanies` (parsed and split)
    - `caseLegalBasisCode`, `caseLegalBasisLabel`
    - `caseLastDecisionDate`, `caseInitiationDate`
    - `decisionLabel`
  - Results are saved/updated in `cases.json`.

- **Fill missing legal basis for mergers**
  - Function `fill_merger_legal_basis(json_file, fill_value="Art. 105")`.
  - For all cases with `caseInstrument == "Merger"` and empty `caseLegalBasisLabel`, sets the label to `["Art. 105"]`.

- **Normalize legal basis labels**
  - Function `normalize_legal_basis(json_file, mapping, save_as=None)`.
  - Uses a mapping (e.g. `"Art. 101 TFEU" → "Art. 101"`, `"Art. 105 TFEU (Ex 85 EC)" → "Art. 105"`) to collapse spelling/format variations.
  - Writes normalized labels back to `cases.json`.

- **Parse NACE-based sector information**
  - Uses `parse_sector_metadata(raw_sector)` plus a `NACE_SECTIONS` dictionary.
  - Splits `caseSectorLabel` into structured metadata:
    - `sectionCode`, `sectionName`
    - `division`, `group`, `classCode`, `classDescription`
  - Adds this under `sector_metadata` in each case in `cases.json`.

- **Convert JSON to CSV**
  - Iterates over `cases.json` and flattens each case into a record:
    - Base fields (`caseId`, `caseInstrument`, `caseNumber`, `caseTitle`, etc.).
    - Sector fields from `sector_metadata`.
    - Joins list fields such as `caseCompanies` and `caseLegalBasisLabel` with `"; "` for CSV.
  - Writes `cases.csv` (UTF‑8).

- **Final CSV cleaning**
  - Replaces empty strings / invalid values with proper nulls.
  - Cleans `caseCompanies` (removing competition terms like `"JV"`, `"KKR"`, `"CVC"`).
  - Saves the final cleaned CSV as `cases.csv`.


Neo4j and the Neo4j Browser / Desktop are required if you want to run the `.cypher` scripts and build the graph.

### Building & Visualizing the Graph in Neo4j Desktop

1. **Start Neo4j Desktop and create an instance**
   - Open Neo4j Desktop.
   - Go to **Data Services → Local Instances**.
   - Click **Create new instance**, give it a **name**, and set a **database user name and password**.
   - Click **Create** and wait until the instance is created and started.

2. **Load the CSV into Neo4j and build the graph**
   - Make sure `cases.csv` is accessible to Neo4j (for file-based import, typically via the Neo4j `import` folder).
   - In Neo4j Desktop, open the **Query** tab for your database.
   - Copy–paste the content of `build_queries.cypher` into the query editor.
   - Adjust the `LOAD CSV` path if needed (e.g., `file:///cases.csv` must match the file location Neo4j sees).
   - Run the script to create the `Case`, `Section`, `LegalBasis`, and `Company` nodes and their relationships.

3. **Visualize the graph**
   - In the same Query tab, paste the content of `visualize_query.cypher`:
     - `MATCH (a)-[r]->(b) RETURN a, r, b;`
   - Run the query to display all nodes and relationships in the Neo4j Browser graph visualization.

### Visualizing & Querying with NeoDash

You can build interactive dashboards on top of the Neo4j graph using **NeoDash using Docker** [`neo4j-labs/neodash`](https://github.com/neo4j-labs/neodash/blob/master/about.md) or you can use Neo Dash  cloud version

1. **Run NeoDash via Docker**
   - Make sure Docker is installed and running.
   - In a terminal, run:

```bash
docker pull neo4jlabs/neodash:latest
docker run -it --rm -p 5005:5005 neo4jlabs/neodash
```

   - NeoDash will be available at `http://localhost:5005`.

2. **Create a dashboard and connect to Neo4j**
   - Open `http://localhost:5005` in your browser.
   - Choose **Create new dashboard**.
   - When prompted to connect, enter the **Bolt URL**, **username**, and **password** of the Neo4j database you created in Neo4j Desktop.

3. **Enable Text2Cypher and use natural language**
   - In the NeoDash UI, open the **Extensions** panel on the right.
   - Add/enable the **Text2Cypher** extension.
   - Provide your **OpenAI API key** when requested.
   - You can now create reports and dashboards by describing what you want in natural language; NeoDash uses Text2Cypher to generate Cypher queries for your competition law graph.

