---
name: Edit a Sparx Enterprise Architect model safely
description: >-
  Create or change elements, attributes, operations, connectors and diagrams in a live Enterprise
  Architect repository, with a baseline captured first so the change can be undone.
api: mcp/sparx-enterprise-architect-mcp.yml
surface: MCP (STDIO)
operations:
  - create_baseline
  - apply_baseline
  - get_root_packages
  - get_packages_information
  - create_or_update_package
  - create_or_update_elements
  - create_or_update_attributes
  - create_or_update_operations
  - create_or_update_connectors
  - create_or_update_messages
  - create_or_update_diagram
  - place_elements_on_diagram
  - remove_elements_from_diagram
  - layout_connectors
  - clone_elements
  - clone_package
  - reload_diagrams
---

# Edit a Sparx Enterprise Architect model safely

## Preconditions

- `MCP3.exe` must have been launched with **`-enableEdit`**. Without it every tool in this skill is
  absent. Do not ask the user to add `-enableDelete` unless deletion is genuinely the request.
- The provider recommends backing up the project before enabling editing at all.
- Consider asking the user to launch with `-modifiedInfoPath <file.csv>`, which appends every
  AI-authored add/edit/delete to a CSV. It is the only way to tell later which changes came from an
  agent rather than a person.

## Steps

1. **Locate the target package.** `get_root_packages`, then `get_packages_information` or
   `find_packages_by_name`. Every write is scoped to a package; get this right before touching
   anything.
2. **Capture a baseline. This step is not optional.** Call `create_baseline` on the package you are
   about to change. It is the only reversal path this product exposes to an agent — `apply_baseline`
   restores the package from it. Tell the user the baseline was taken and that you can roll back.
3. **Get the type string right before creating.** Element `type` must match the Enterprise Architect
   Automation Interface type name exactly. The server publishes MCP prompts for this —
   *Creation Rules for UML*, *Creation Rules for SysML 1.5*, *Creation Rules for BPMN 2.0*. Load the
   prompt for the notation in use rather than guessing a type name. For other notations such as
   ArchiMate, ask the user for the type strings; there is no shipped prompt.
4. **Create structure before visuals.** `create_or_update_package` → `create_or_update_elements` →
   `create_or_update_attributes` / `create_or_update_operations` → `create_or_update_connectors`.
   Then `create_or_update_diagram` and `place_elements_on_diagram`.
   In `place_elements_on_diagram`, position and size may be omitted to keep current values.
5. **Tidy the layout last** with `layout_connectors`, then `reload_diagrams` so the user sees the
   result in Enterprise Architect.
6. **Verify what you wrote.** Read it back with `get_elements_information` /
   `get_connectors_information`. Several documented defects have been of the "value silently not
   applied" kind (connector routeStyle, width of 1, isReference on Parts, parameter ordering), so a
   successful call is not proof the value landed.

## Rules

- **Baseline first, always.** No baseline, no edit.
- **Never enable or use deletion casually.** `delete_from_model` and `delete_taggedvalue_from_model`
  need `-enableDelete`, and the provider reports that during their own testing an instruction to
  delete a package's tagged values deleted the **entire package**. If deletion is genuinely
  required, take a baseline, confirm the exact scope with the user in writing, then delete.
- **There is no idempotency.** No idempotency key exists on any surface. If a call's outcome is
  unclear, read the model back — do not re-issue the create.
- **Sequence diagrams are order-sensitive.** Message ordering was reworked across v2.8.5–v2.8.11; do
  not rely on a returned message `order` value, and re-read after adding or deleting messages.
- **Report the reversal path in your final message**, naming the baseline you took, so the user
  knows exactly how to undo the work.

## See also

- Reversibility block: `conventions/sparx-enterprise-architect-conventions.yml`
- Known failure modes: `errors/sparx-enterprise-architect-problem-types.yml`
- Release notes: `changelog/sparx-enterprise-architect-changelog.yml`
