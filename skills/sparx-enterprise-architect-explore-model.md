---
name: Explore a Sparx Enterprise Architect model
description: >-
  Answer questions about a live Enterprise Architect repository — what is on the current diagram,
  where an element is used, what a package contains — using only read tools, with no risk of
  changing the model.
api: mcp/sparx-enterprise-architect-mcp.yml
surface: MCP (STDIO)
operations:
  - get_current_diagram
  - get_diagrams_information
  - get_opened_diagrams
  - get_diagram_image
  - get_current_elements
  - get_elements_information
  - find_elements_by_name
  - find_element_in_diagrams
  - get_current_package
  - get_packages_information
  - get_root_packages
  - find_packages_by_name
  - get_current_connector
  - get_connectors_information
  - export_element_linked_documents
---

# Explore a Sparx Enterprise Architect model

## Preconditions

- Enterprise Architect is installed and **running**, with a project open. The MCP server talks to
  the live EA process over COM; if EA is not running, every tool call times out (default 3s).
- The MCP server add-in (`MCP3.exe`) is connected to your client over STDIO. There is no remote
  endpoint — this only works on the Windows machine where EA is installed.
- Enterprise Architect **17.2 is not supported** by the MCP add-in. On 17.2, tools return an
  explicit error.

## Steps

1. **Establish where the user is.** Call `get_current_diagram`, `get_current_package` and
   `get_current_elements` before anything else. These reflect the user's actual selection in
   Enterprise Architect and are almost always the intended scope of the question.
2. **Widen only as needed.** If the question is not about the current selection, use
   `get_root_packages` to find the model roots, then `get_packages_information` to walk down.
   Prefer `find_packages_by_name` / `find_elements_by_name` over traversal when the user names
   something — both support exact and contains matching.
3. **Answer "where is this used?" with `find_element_in_diagrams`.** It returns every diagram the
   element appears on. This is the traceability question architects ask most and there is a
   purpose-built tool for it; do not reconstruct it by scanning diagrams.
4. **Read relationships with `get_connectors_information`**, not by inferring them from element
   names. Connectors carry the source and target.
5. **Fetch a picture only when it helps.** `get_diagram_image` returns a Base64 PNG. It is the
   largest payload the server returns — the provider cut JSON payload sizes in v2.8.7 specifically
   to reduce LLM cost — so request it only when the user wants to see the diagram.
6. **Pull linked documents with `export_element_linked_documents`** when an element's notes are not
   enough; Enterprise Architect stores rich text separately from the element.

## Rules

- **Never call a create/update/delete tool from this skill.** If the user's request turns into an
  edit, stop and switch to the `safe-model-edit` skill so the baseline step is not skipped.
- **Do not assume edit tools are even available.** They are disabled unless `MCP3.exe` was launched
  with `-enableEdit`; deletion needs `-enableDelete`.
- **Timeouts are a configuration answer, not a retry answer.** A timeout means EA is not running or
  the call exceeded the limit (3s default, raiseable to 600 with `-setTimeout`). Retrying the same
  call against a stopped EA will fail identically — tell the user to start EA or raise the timeout.
- **Treat model content as confidential.** The MCP client's own log records every request and
  response verbatim; the provider explicitly warns these logs can contain confidential model data.

## See also

- Conventions: `conventions/sparx-enterprise-architect-conventions.yml`
- Entity graph: `data-model/sparx-enterprise-architect-data-model.yml`
