# Query Connect: Talking to Databases Like a Human

Built this thing called Query Connect where you can ask questions about your data in plain English and get back actual SQL queries and charts. Uses BigQuery, Vertex AI's Gemini, and Streamlit.

## The Idea

Most people don't know SQL. But everyone has questions about their data. "What were our top products last month?" or "Which customers are buying the most?"

Query Connect lets you just ask those questions naturally. It figures out the SQL, runs it, and shows you the results with charts. No SQL knowledge needed.

## The Stack

Kept it simple:
- **BigQuery**: Where the e-commerce data lives
- **Gemini (Vertex AI)**: Converts English to SQL
- **Streamlit**: The web interface
- **Plotly**: For the charts

Nothing exotic, just solid tools that work.

## How It Actually Works

The core is function calling with Gemini. You define functions that the LLM can use, and it decides when to call them.

I set up two main functions:

```python
list_datasets_func = FunctionDeclaration(
    name="list_datasets",
    description="Get a list of datasets that will help answer the user's question",
    parameters={
        "type": "object",
        "properties": {},
    },
)

sql_query_func = FunctionDeclaration(
    name="sql_query",
    description="Get information from data in BigQuery using SQL queries",
    parameters={
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "SQL query to answer the user's question"
            }
        },
        "required": ["query"],
    },
)
```

Simple, but it works. Gemini looks at the user's question, decides which function to call, and generates the SQL.

## Automatic Chart Selection

One thing I'm proud of: the system picks the right chart type automatically based on what data comes back.

```python
def determine_visualization_type(query_results: List[Dict[str, Any]]) -> str:
    df = pd.DataFrame(query_results)

    # Got dates? Line chart
    temporal_columns = df.select_dtypes(include=['datetime64']).columns
    if len(temporal_columns) > 0:
        return "time_series"

    # Got lat/long? Map
    if all(col in df.columns for col in ['latitude', 'longitude']):
        return "geographic"

    # Multiple numbers? Probably a correlation
    numerical_columns = df.select_dtypes(include=[np.number]).columns
    if len(numerical_columns) >= 2:
        return "correlation"
    elif len(numerical_columns) == 1:
        return "distribution"

    # Otherwise just compare categories
    return "comparison"
```

It's not perfect, but catches most common cases. Time series get line charts, distributions get histograms, etc.

## The Flow

When you ask a question:
1. Gemini reads it and figures out what SQL to write
2. Query runs on BigQuery (with safety limits)
3. Results come back
4. System picks the right chart type
5. Plotly generates the visualization

Takes maybe 2-3 seconds end-to-end.

## Data Quality Stuff

I added some basic data quality checks because bad data in = bad answers out:

```python
def check_data_quality(table_id: str, columns: str) -> Dict[str, Any]:
    quality_checks = {}
    for column in columns_list:
        query = f"""
        SELECT
            COUNT(*) as total_rows,
            COUNT(DISTINCT {column}) as unique_values,
            COUNT(CASE WHEN {column} IS NULL THEN 1 END) as null_count,
            COUNT(CASE WHEN TRIM({column}) = '' THEN 1 END) as empty_string_count
        FROM `{table_id}`
        """
        results = client.query(query).to_dataframe()
        quality_checks[column] = results.to_dict('records')[0]

    return quality_checks
```

Checks for nulls, empty strings, distinct values - basic stuff but catches issues early.

## Making the Charts

Plotly handles the visualization:

```python
def generate_visualization(query_results: str, viz_type: str, title: str) -> go.Figure:
    df = pd.DataFrame(eval(query_results))

    if viz_type == "time_series":
        fig = px.line(df, x=df.columns[0], y=df.columns[1], title=title)
    elif viz_type == "distribution":
        fig = px.histogram(df, x=df.columns[0], title=title)
    # More chart types...

    return fig
```

Pretty straightforward. Plotly makes this easy.

## Security

Not trying to be fancy here, just sensible defaults:

```python
credentials_dict = {
    "type": "service_account",
    "project_id": st.secrets["gcp_project_id"],
    "private_key_id": st.secrets["gcp_private_key_id"],
    "private_key": st.secrets["gcp_private_key"],
    # More creds...
}
```

Secrets stay in Streamlit's secrets file, never in code.

Also set query limits in BigQuery so nobody can accidentally run a $10,000 query. Important!

## The Interface

Kept it simple with three tabs:

1. **Dataset Overview**: What data you have
2. **Example Questions**: Pre-written queries you can click
3. **Chat**: Where you type questions

```python
tab1, tab2, tab3 = st.tabs(["Dataset Overview", "Available Data", "Example Questions"])

with tab1:
    st.markdown("""
        E-commerce dataset with:

        | Category | Description |
        |----------|-------------|
        | 🛍️ Products & Orders | Order history, product details |
        | 👥 Customer Behavior | Demographics, purchase patterns |
        | 📈 Sales Metrics | Revenue, margins |
        | 🔍 Product Analytics | Categories, inventory |
    """)
```

Example questions are clutch. Most users don't know what to ask at first, so having examples gets them started.

## Performance Tricks

A few things to keep it fast:

- Set max bytes billed in BigQuery (safety + speed)
- Cache results in session state
- Timeout long-running queries
- Limit result sizes

Nothing revolutionary, just don't let users shoot themselves in the foot.

## Deployment

Wrote a bash script to set everything up:

```bash
#!/usr/bin/env bash
# Enable services
gcloud services enable aiplatform.googleapis.com
gcloud services enable bigquery.googleapis.com

# Create dataset
bq mk --force=true --dataset thelook_ecommerce
# More setup...
```

Then just `streamlit run app.py`. Easy.

## What I'd Change

Some things that could be better:

**Context awareness**: Right now each question is independent. Would be cool if it remembered previous queries ("Now show me last year" should work).

**Query refinement**: If the generated SQL is wrong, there's no easy way to tweak it. Should add that.

**Better error messages**: When queries fail, the errors are... not great. Need to make those more helpful.

**Caching**: Could cache common queries to make repeat questions instant.

## Lessons Learned

Building this taught me a few things:

**LLMs are pretty good at SQL**: Gemini nails the SQL generation like 80% of the time. The other 20% is usually edge cases or ambiguous questions.

**Example queries matter**: Users need guidance. Nobody wants to stare at a blank text box.

**Visualization type matters more than I thought**: A bad chart can make good data useless. Auto-selecting the right one is worth the effort.

**Cost controls are not optional**: Set those BigQuery limits from day one. Trust me.

## Final Thoughts

This was a fun project. Natural language to SQL isn't a new idea, but LLMs make it actually work reliably now.

The key is keeping it simple. You don't need fancy prompt engineering or complex systems. Just good function definitions and sensible defaults.

If you're building something similar, start with the basics. Get the SQL generation working first. Everything else is polish.

Code's on GitHub if you want to check it out or build on it. Happy to answer questions!