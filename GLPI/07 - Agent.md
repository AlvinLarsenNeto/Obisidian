---
title: GLPI Agent
source: https://glpi-agent.readthedocs.io/en/1.12/
updated: 2026-04-20
status: oficial
area: glpi
tags: [glpi, agent, inventario, deploy]
---

# GLPI Agent

## Fontes oficiais
- https://glpi-agent.readthedocs.io/en/1.12/
- Man page: https://glpi-agent.readthedocs.io/en/nightly/man/glpi-agent.html

## Eixos principais da documentação
- Installation
- Configuration / Settings
- Usage / Execution mode
- Tasks
- HTTP Interface
- Plugins
- Bug reporting
- Man pages

## Pontos relevantes
- O agente é multiplataforma (Windows, Linux, macOS)
- Pode operar standalone ou centralizado
- Executa inventário local, descoberta de rede, deployment e outras tarefas
- **Fusion Inventory não é mais suportado** nas versões atuais do GLPI

## Estrutura de notas sugeridas
- [ ] GLPI Agent — Instalação Windows (MSI/silencioso)
- [ ] GLPI Agent — Instalação Linux
- [ ] GLPI Agent — Configuração (`agent.cfg`)
- [ ] GLPI Agent — Execução e agendamento (service, tarefa Windows)
- [ ] GLPI Agent — Inventário (campos coletados, frequência)
- [ ] GLPI Agent — HTTP Interface (porta 62354, forçar inventário)
- [ ] GLPI Agent — Troubleshooting (logs, inventário não chegando)
