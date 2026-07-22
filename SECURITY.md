# Security policy

## Supported deployment

This repository is intended for self-hosted Coolify deployments. The default Compose configuration keeps the API on loopback and does not publish PostgreSQL or Redis.

## Secret handling

Never report or commit:

- DeepSeek or OpenAI API keys;
- PostgreSQL passwords;
- Honcho JWT secrets;
- Hermes/OpenClaw OAuth credentials;
- database dumps or memory data;
- Coolify access tokens.

Configure secrets in Coolify's environment-variable interface. Use `.env.example` only as a variable-name reference.

## Reporting a vulnerability

Please do not open a public issue for a suspected security vulnerability. Contact the repository owner privately through GitHub with:

- a concise description;
- affected file or deployment condition;
- reproduction steps that do not include secrets;
- suggested mitigation, if available.

Allow reasonable time for assessment and remediation before public disclosure.
