# External API (IQ)

Interactive Swagger UI for the PressurePro IQ External API.

**Source of truth:** [`phx-infrastructure` `services/External-API/openapi.yaml`](https://github.com/PressurePro/phx-infrastructure/blob/PHX-705-ExternalAPI/services/External-API/openapi.yaml)  
**Base URLs:** stage `https://stg.api.pressurepro.us` · prod `https://prd.api.pressurepro.us`  
**Auth:** `Authorization: Bearer <external-api-key>` (Phoenix Rapid external API keys)

This page documents **live** gateway routes only. Try “Authorize” in Swagger UI with a stage/prod external key to exercise requests from the browser (CORS permitting).

<swagger-ui src="./openapi.yaml"/>
