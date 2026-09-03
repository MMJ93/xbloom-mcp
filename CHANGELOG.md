# Changelog

## 2.1.0 - 2026-09-03

- Add `xbloom_list_shared_recipes` using xBloom's account-shared endpoint.
- Request up to 100 shared recipes so the API does not silently return only
  the first six records.
- Return recipe IDs, brew parameters, and share links.
- Derive the OAuth issuer/resource URL from the active Supabase project so
  forks do not point back to the upstream deployment.
- Identify the MCP server and health response as version 2.1.0.
