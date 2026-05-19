---
title: GLPI - CLI
source: https://help.glpi-project.org/documentation/cli
updated: 2026-04-20
status: oficial
area: glpi
tags: [glpi, cli, console, comandos]
---

# GLPI — CLI

## Fonte oficial
- https://help.glpi-project.org/documentation/cli

## Resumo
O GLPI possui interface de linha de comando via `bin/console`, executada a partir da raiz da instalação.

```bash
php bin/console <comando>
```

## Categorias de comandos

### Assets
```bash
glpi:assets:cleansoftware    # limpa software duplicado
glpi:assets:purgesoftware    # purga software inativo
```

### Build
```bash
glpi:build:compile_scss      # recompila estilos
```

### Manutenção
- Comandos de limpeza e manutenção preventiva

### Banco de dados
- Migrations, check de integridade, atualização de schema

### Usuários
- Criação, reset de senha, sincronização LDAP

### Tarefas automáticas
- Forçar execução de cron jobs específicos

### Debug / Diagnóstico
- Verificação de configuração, logs, status de plugins

### Migração
- Comandos específicos para upgrade entre versões (ex: GLPI 10 → 11)

## Notas para expandir
- [ ] Listar todos os comandos relevantes para a operação ASV
- [ ] Documentar como forçar a execução de `SlaTicketEscalate`
- [ ] Documentar reset de senha via CLI para emergências
