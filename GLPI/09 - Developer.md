---
title: GLPI - Developer
source: https://glpi-developer-documentation.readthedocs.io/en/master/
updated: 2026-04-20
status: oficial
area: glpi
tags: [glpi, developer, api, plugins]
---

# GLPI — Developer

## Fontes oficiais
- Documentação de desenvolvedor: https://glpi-developer-documentation.readthedocs.io/en/master/
- API do desenvolvedor: https://glpi-developer-documentation.readthedocs.io/en/master/devapi/index.html
- High-Level API (GLPI 11): https://glpi-developer-documentation.readthedocs.io/en/master/devapi/hlapi/index.html
- Tutorial de plugins: https://glpi-developer-documentation.readthedocs.io/en/master/plugins/tutorial.html

## Áreas principais
- Source code management
- Coding standards
- Developer API (Low-level)
- High-Level API ← novo no GLPI 11
- Plugins Guidelines
- Packaging
- Upgrade guides

## High-Level API (GLPI 11)
Nova API REST apresentada no GLPI 11 — mais limpa e padronizada que a API legada.  
Relevante para integração com o Portal ASV (edge functions que consultam o GLPI diretamente).

## Notas para expandir
- [ ] Autenticação na API (session token, app token)
- [ ] Endpoints principais: tickets, users, entities
- [ ] Criação de ticket via API (usado em edge functions ASV)
- [ ] Consulta de SLA e campos de prazo via API
- [ ] Diferenças API v1 (GLPI 10) vs High-Level API (GLPI 11)
