## Keyword & Phrase Discovery Tool
## Purpose

This tool was developed to support maintenance of the Signal Interpretation Layer by identifying recurring keywords, phrases, and contextual variations within historical consumer communications.

## The tool assists with:
- Keyword extraction
- Bigram extraction
- Trigram extraction
- Anchor-driven phrase expansion
- Signal discovery
- Signal refinement
- Continuous improvement activities

## Why It Was Created

As correspondence volumes increased, manually identifying emerging phrases and contextual variations became increasingly difficult.

This tool was designed to help:
- Discover previously unidentified signals
- Expand category coverage
- Improve consistency of signal detection
- Support audit and governance activities

## Key Features
### Keyword Extraction

Identifies frequently occurring keywords while excluding stop words and non-relevant tokens.

### Phrase Extraction

Extracts:

- Two-word phrases (Bigrams)
- Three-word phrases (Trigrams)
- Anchor-Driven Expansion

Allows analysts to define anchor phrases and automatically identify related contextual variations.

Example:
Anchor Phrase:
contact restriction

Expanded Phrases:
full contact restriction
contact restriction request
consumer contact restriction

## Data Filtering
Automatically excludes:
- Dates
- Times
- Currency values
- URLs
- Email addresses
- Configurable stop words
- Output

## The tool generates:
- Ranked keywords
- Ranked bigrams
- Ranked trigrams
- Expanded anchor phrases
- Frequency counts
- Typical Use Cases
- Signal discovery
- Category enhancement
- Keyword maintenance
- Audit preparation
- Framework refinement

## Technical Notes
Developed in VBA and designed to operate within Microsoft Excel environments where external tooling may be restricted.

## Source Code
The VBA source code is available here: Anchor-Driven-Phrase-Expansion.bas
