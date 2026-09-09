# External API (IQ)

Interactive Swagger UI for the PressurePro IQ External API.

**Base URL:** `https://prd.api.pressurepro.us`  
**Auth:** `Authorization: Bearer <external-api-key>`

Use **Authorize** in Swagger UI with your external API key to try requests from the browser (CORS permitting). PressurePro will provide your API key directly.

For a Python walkthrough of ingesting TPMS readings, see [Sending TPMS Readings to IQ (Python)](../../technical-docs/ingest-sensor-readings-python.md).

<swagger-ui src="./openapi.yaml"/>
