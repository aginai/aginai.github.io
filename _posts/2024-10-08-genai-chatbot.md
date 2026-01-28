# Building a Chatbot That Actually Knows Your Docs

So I built this RAG (Retrieval-Augmented Generation) system using Azure and Streamlit. The goal was simple: make a chatbot that could answer questions about specific documents without hallucinating stuff.

You know how regular LLMs will just confidently make things up sometimes? RAG fixes that by making the model look up information first before answering. It's like the difference between answering from memory versus checking your notes first.

## Why RAG?

The problem with just throwing documents at an LLM is that they'll either hallucinate answers or tell you they don't know. With RAG, you:
1. First search your documents for relevant info
2. Then hand that info to the LLM
3. The LLM generates an answer based on what it found

Way more reliable. And you can actually cite your sources.

## How It Works

The system has three main pieces:

**Document Processing**: Takes your PDFs, breaks them into chunks, extracts text (even from tables), and uploads everything to Azure. This part was trickier than I expected - PDFs are messy.

**Azure Cognitive Search**: Indexes everything so you can search through it fast. Azure's semantic search is actually pretty good for this.

**Streamlit Frontend**: Just a clean chat interface. Nothing fancy, but it works well. Added persona selection so different user types could get different response styles.

## The PDF Processing Headache

PDFs are annoying to work with. Some use native text, some are scanned images, some have complex tables. I ended up with two approaches:

```python
def get_document_text(filename):
    if localpdfparser:
        # Simple approach: use PyPDF
        reader = PdfReader(filename)
        pages = reader.pages
        for page_num, p in enumerate(pages):
            page_text = p.extract_text()
            page_map.append((page_num, offset, page_text))
    else:
        # Better approach: Azure Form Recognizer
        # Handles tables and complex layouts way better
        form_recognizer_client = DocumentAnalysisClient(...)
```

For tables, I convert them to HTML so they stay readable:

```python
def table_to_html(table):
    table_html = "<table>"
    rows = [sorted([cell for cell in table.cells if cell.row_index == i],
            key=lambda cell: cell.column_index)
            for i in range(table.row_count)]
    # Build the HTML structure...
    return table_html
```

## Text Chunking Strategy

One thing I learned: chunk size matters a lot. Too small and you lose context. Too big and your search gets less precise.

I ended up with:
- Overlapping chunks (so context isn't lost at boundaries)
- Respect sentence boundaries (don't cut mid-sentence)
- Special handling for tables that span multiple chunks

It's not perfect, but it works pretty well in practice.

## Setting Up Azure Search

The search index setup is straightforward:

```python
search_index = SearchIndex(
    name=index,
    fields=[
        SimpleField(name="id", type="Edm.String", key=True),
        SearchableField(name="content", type="Edm.String", analyzer_name="en.microsoft"),
        SimpleField(name="category", type="Edm.String", filterable=True, facetable=True),
        # More fields...
    ],
    semantic_settings=SemanticSettings(...)
)
```

Semantic search makes a huge difference here. It understands that "machine learning" and "ML" are related, which keyword search wouldn't catch.

## The Interface

Kept the UI simple with Streamlit. You can:
- Pick a persona (different roles get different response styles)
- Ask questions in plain English
- See your chat history
- Get answers with source citations

The persona thing was interesting. Turns out engineers and managers ask questions differently and want different levels of detail.

## Running It

Setup is pretty minimal:

```toml
# secrets.toml
[default]
searchservice = "your-search-service"
searchkey = "your-api-key"
index = "your-index-name"
```

Then just:
```bash
streamlit run app.py
```

## Things I'd Do Differently

A few pain points I hit:

**PDF handling**: Should've just used Azure Form Recognizer from the start. The PyPDF fallback works but misses stuff.

**Chunk size**: Took way too long to tune this. Probably should've just started with 1000 tokens with 200 token overlap.

**Search tuning**: The default search settings were actually fine. Spent too much time over-optimizing.

**Prompt engineering**: This mattered more than I expected. Getting the LLM to cite sources properly took some iteration.

## What's Next

If I revisit this, I'd add:
- Support for more document types (Word docs, Excel, etc.)
- Better conversation context handling (right now it's pretty basic)
- Custom embeddings instead of relying on Azure's default ones
- Some way to update docs in real-time without reprocessing everything

## Final Thoughts

This was a fun project. RAG isn't magic - it's just good search plus LLMs. But when done right, it's way more useful than either alone.

The key insight: don't over-engineer it. Start simple, see what breaks, then fix that. Most of the value comes from just getting it working, not from fancy optimizations.

If you want to build something similar, Azure makes it pretty easy. The hard part isn't the tech - it's figuring out how to chunk and index your docs well.