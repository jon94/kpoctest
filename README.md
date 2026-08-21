# kpoctest

Proof-of-concept workspace.
restate + powersync metrics

# Export all Restate monitors via API
curl -s "https://api.datadoghq.com/api/v1/monitor?tag=service:restate" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | jq . > restate_monitors.json