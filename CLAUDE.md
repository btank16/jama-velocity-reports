# CLAUDE.md - Project Guide

## Project Overview
Jama Velocity Reports — a collection of Velocity Template Language (.vm) report templates for **Jama Connect** (requirements/testing management platform). These templates generate HTML/Word-format reports from Jama project data.

## Project Structure
```
├── DIR Export.vm          # Directory/DIB report template
├── Risk Export.vm         # Risk assessment report template
├── Use Task Export.vm     # Use task/scenario report template
├── User Needs Export.vm   # User needs/requirements report template
├── URRA Export.vm         # User Risk and Requirements Analysis report template
├── Data Dictionary.xlsx   # Data mapping reference
└── README.md
```

## Technology
- **Language**: Velocity Template Language (VTL) — `.vm` files
- **Platform**: Jama Connect Reporting Engine
- **Output**: MIME multipart format (Word .doc / Excel .xls embedding via HTML)
- **Data Access**: Jama DAOs (DocumentDao, AttachmentDao, RelationshipDao, etc.)

## Reference Documentation
- When working on templates, use WebFetch to consult the Jama Velocity documentation: https://velocity.jamasoftware.com/latest/ProxiedVelocity-9-5/

## Key Patterns
- Templates use `#set`, `#foreach`, `#if`/`#else` Velocity directives
- Data is accessed through Jama-provided context variables and DAO objects
- Each template outputs a complete MIME multipart document with embedded CSS/HTML
- Reports support both document and workbook output formats
- Field access typically follows: `$item.getFieldValue("fieldName")`

## Conventions
- Template files are named `<Report Name> Export.vm`
- No build system — templates are uploaded directly to Jama Connect
- Commit messages are lowercase, concise descriptions of changes
