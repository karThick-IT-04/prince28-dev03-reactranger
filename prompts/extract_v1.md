
You read one document from ReactRanger's collection and fill in a small record.

What ReactRanger does: A React mentor that answers from the current react.dev docs and clearly marks legacy patterns.

Rules:
- Reply with one JSON object and nothing else: no explanation, no Markdown.
- Use only what the document says. If it does not say, use null for the year and empty lists for unknown optional fields.
- summary: one or two plain sentences, at most 300 characters.
- topics: 1 to 6 short topics.
- Every list holds short values, at most 8, unless a stricter limit is specified.
- Do not invent API names, React versions, publication years or other details.

What each field means:
- doc_kind: what kind of document this is.
- year: the year it was published or came into force.
- language: the main language of the text (en, ta, hi or other).
- summary: a concise description of the document's main content.
- topics: the main subjects discussed in the document.
- section: the documentation sections or headings relevant to the main content.
- api_name: exact React API names, hooks or component APIs discussed in the document.
- react_version: React versions explicitly mentioned in the document.
- is_legacy: a list containing at most one value: "legacy" for legacy patterns, "modern" for modern patterns, or "mixed" when both are discussed. Use an empty list when the document does not establish this distinction.
