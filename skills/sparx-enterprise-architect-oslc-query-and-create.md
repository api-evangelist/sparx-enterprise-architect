---
name: Query and create Enterprise Architect resources over OSLC
description: >-
  Integrate with a remote Enterprise Architect repository over the Pro Cloud Server OSLC
  Architecture Management 2.0 RESTful API — authenticate, discover services, query the model, and
  create a requirement.
api: authentication/sparx-enterprise-architect-authentication.yml
surface: HTTP (Pro Cloud Server OSLC)
operations:
  - POST /{model_name}/oslc/am/login/
  - GET /{model_name}/oslc/am/sp/
  - GET /{model_name}/oslc/qc/
  - POST /{model_name}/oslc/cf/
  - GET /{model_name}/oslc/re/{GUID}/
  - GET /{model_name}/oslc/am/logout/
---

# Query and create Enterprise Architect resources over OSLC

## Preconditions

- A **Pro Cloud Server with a paid licence**. Secure cloud connections and floating licences work on
  the free edition; OSLC does not.
- The base URL is the customer's own server, not a Sparx host:
  `<protocol>://<server>/<model_name>/oslc/`. The documentation's own example is
  `http://localhost:480/firebird_model/oslc/`. Ask the user for the host and model name; never guess
  one.
- Everything on the wire is **RDF/XML**. There is no JSON representation.

## Steps

1. **Authenticate and keep the token.** POST to `/{model_name}/oslc/am/login/` with the body that
   matches the project's security mode:
   - model security enabled → `uid=<USER ID>;pwd=<PASSWORD>;`
   - Windows SSO → `sso=ntlm;`
   - OpenID Connect → `sso=openid;code=<AUTHORIZATION CODE>;redirecturi=<REDIRECT URI>;`
   - security disabled → `uid=;pwd=;` (a token is still required)

   The RDF response carries the token as `ss:useridentifier`. Pass it as `useridentifier` on every
   subsequent request; without it Pro Cloud Server rejects the call outright.

2. **Read the capability flags in the same response before doing anything else.** `ss:validlicense`
   tells you whether OSLC is even licensed, `ss:readonlymodel` whether writes are possible at all,
   and `ss:elementpermission` / `ss:diagrampermission` what this user may change. Checking these
   costs nothing and turns three separate late failures into one early, clear answer.

3. **Send the access code if the project has one.** When configured, every request must carry the
   custom header `EAO-Access-Code: <code>`. It is validated in addition to, not instead of, the
   token.

4. **Discover the services.** `GET /{model_name}/oslc/am/sp/` returns the Service Provider Resource:
   the URL to POST to for creation, the URL to GET existing resources, and the resource-shape
   metadata describing both the creation payload and the resource representation. Read the shapes
   rather than assuming property names.

5. **Query the model.** `GET /{model_name}/oslc/qc/`, narrowing with `oslc.where` and trimming the
   payload with `oslc.select` / `oslc.properties` (declare prefixes with `oslc.prefix`). Each
   returned resource carries `rdf:about` = `/{model_name}/oslc/re/<GUID>/`, its addressable URL.

   **There is no paging.** No page-size, offset or continuation parameter is documented. On a large
   repository the only way to bound a result set is a tighter `oslc.where`. Budget for that.

6. **Resolve a packageID before creating anything.** The Creation Factory requires `ss:packageID`
   and the package must already exist. Get it from step 5; do not construct one.

7. **Create with the Creation Factory.** POST RDF/XML to `/{model_name}/oslc/cf/`, for example an
   `oslc_rm:requirement` with `dcterms:title` (required), `ss:packageID` (required), and optionally
   `dcterms:description`, `dcterms:creator`, `ss:type`, `ss:difficulty`, `ss:priority`. On success
   the response carries the URL of the new resource.

8. **Log out** with `GET /{model_name}/oslc/am/logout/?useridentifier=<token>` when finished.

## Rules

- **Do not send read-only properties.** `identifier`, `created` and `modified` are reference-only.
- **Omit a property rather than sending it empty.** An empty constrained value is an error, not a
  no-op — the documented example is an empty `ss:difficulty`.
- **Assume no retry safety.** No idempotency key exists. A retried Creation Factory POST creates a
  second element. Query before you re-POST.
- **Understand the audit gap before writing.** The provider states that changes made through OSLC
  are **not recorded even when Enterprise Architect auditing is enabled**, and no reversal operation
  is published for this surface. An agent writing over OSLC leaves no vendor-side trail and has no
  undo. If reversibility matters, do the write through the MCP server with a baseline instead, or
  arrange a repository backup first — and say so to the user before acting.

## See also

- Auth detail: `authentication/sparx-enterprise-architect-authentication.yml`
- Conventions and reversibility: `conventions/sparx-enterprise-architect-conventions.yml`
- Errors: `errors/sparx-enterprise-architect-problem-types.yml`
- Entities: `data-model/sparx-enterprise-architect-data-model.yml`
