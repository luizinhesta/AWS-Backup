# Testes — Projeto 04 (Global Database & Backup)

> Passo a passo **enxuto** de como executar cada teste do projeto, com o contexto mínimo para seguir. Para a implantação (criação dos recursos), veja `IMPLANTACAO.md`.
>
> **Regiões:** origem **`us-east-1`** (tudo); réplica da Global Table **`us-east-2`**.
> **Comandos:** rodar no **AWS CloudShell** (ícone `>_` no topo do Console) — já autenticado, shell Linux/bash. Cada comando traz o `--region` certo.
> **Princípio da série:** o status "Ativo" **não** é prova (Req. 14.4) — todo teste é comprovado por **operação real**.

---

## Global Tables

### T4G.0 — Réplica ativa nas duas regiões
- **Prova:** a réplica existe.
- **Como:** DynamoDB (`us-east-1`) → **Tabelas** → `carrinho-lab` → aba **Tabelas globais**.
- **Esperado:** `us-east-1` e `us-east-2` com status **Ativo**.

### T4G.1 — Replicação origem → réplica (Multi-Region)
- **Prova:** o que grava em `us-east-1` aparece em `us-east-2`.
- **Como:**
  ```bash
  aws dynamodb put-item --region us-east-1 --table-name carrinho-lab \
    --item '{"cliente_id":{"S":"cliente-teste-44"},"produto_id":{"S":"produto-e1-e2"},"quantidade":{"N":"2"},"nome_produto":{"S":"Gravado em us-east-1"},"preco":{"N":"99"},"atualizado_em":{"S":"2024-01-01T13:00:00Z"}}'
  sleep 5
  aws dynamodb get-item --region us-east-2 --table-name carrinho-lab \
    --key '{"cliente_id":{"S":"cliente-teste-44"},"produto_id":{"S":"produto-e1-e2"}}'
  ```
- **Esperado:** o `get-item` em `us-east-2` retorna o `Item`. (Se não vier de primeira, aguarde e repita — replicação é eventual.)

### T4G.2 — Replicação réplica → origem (Multi-Active)
- **Prova:** a 2ª região também **escreve** e replica de volta (não existe "região primária").
- **Como:**
  ```bash
  aws dynamodb put-item --region us-east-2 --table-name carrinho-lab \
    --item '{"cliente_id":{"S":"cliente-teste-44"},"produto_id":{"S":"produto-e2-e1"},"quantidade":{"N":"3"},"nome_produto":{"S":"Gravado em us-east-2"},"preco":{"N":"150"},"atualizado_em":{"S":"2024-01-01T13:05:00Z"}}'
  sleep 5
  aws dynamodb get-item --region us-east-1 --table-name carrinho-lab \
    --key '{"cliente_id":{"S":"cliente-teste-44"},"produto_id":{"S":"produto-e2-e1"}}'
  ```
- **Esperado:** o `get-item` em `us-east-1` retorna o `Item` gravado em `us-east-2`.

### T4G.3 — Métrica `ReplicationLatency`
- **Prova:** a replicação é mensurável (saúde/atraso).
- **Como:** CloudWatch (`us-east-1`) → **Métricas** → **Todas as métricas** → namespace **`AWS/DynamoDB`** → grupo com `ReceivingRegion` → marcar **`ReplicationLatency`** (`TableName=carrinho-lab`, `ReceivingRegion=us-east-2`). Período 1 min, estatística Média/Máximo.
- **Esperado:** pontos no gráfico, tipicamente de centenas de ms a ~1s, logo após os `put-item`.

> **Limpeza (Global Tables):**
> ```bash
> aws dynamodb delete-item --region us-east-1 --table-name carrinho-lab --key '{"cliente_id":{"S":"cliente-teste-44"},"produto_id":{"S":"produto-e1-e2"}}'
> aws dynamodb delete-item --region us-east-1 --table-name carrinho-lab --key '{"cliente_id":{"S":"cliente-teste-44"},"produto_id":{"S":"produto-e2-e1"}}'
> ```

> **Conceito:** Global Tables = Multi-Region + Multi-Active + replicação assíncrona (consistência eventual; conflito = *last writer wins*). Difere de Multi-AZ (disponibilidade na região) e Read Replica (escala de leitura na região).

---

## AWS Backup — ciclo BACKUP → VALIDAR → RESTORE → VALIDAR DADOS

> Tudo em **`us-east-1`** (AWS Backup para DynamoDB é regional). O teste **não** termina ao criar o backup — só termina em "validar dados".

### T4B.0 — Estrutura de backup criada
- **Prova:** Vault + Plan/Rule + seleção apontando para `carrinho-lab`.
- **Como:** AWS Backup (`us-east-1`): conferir Vault `db-lab-04-vault` (0 pontos), Plano `db-lab-04-plano` com regra `db-lab-04-regra-diaria` (retenção 7 dias) e atribuição `db-lab-04-carrinho` → `carrinho-lab` (role `AWSBackupDefaultServiceRole`).
- **Esperado:** estrutura criada, **sem** Recovery Point ainda.

### T4B.1 — Preparar dados conhecidos (pré-condição)
- **Prova:** há itens estáveis para comparar depois.
- **Como:**
  ```bash
  aws dynamodb put-item --region us-east-1 --table-name carrinho-lab \
    --item '{"cliente_id":{"S":"cliente-backup-46"},"produto_id":{"S":"produto-notebook"},"quantidade":{"N":"1"},"nome_produto":{"S":"Notebook"},"preco":{"N":"3500"},"atualizado_em":{"S":"2024-01-01T14:00:00Z"}}'
  aws dynamodb put-item --region us-east-1 --table-name carrinho-lab \
    --item '{"cliente_id":{"S":"cliente-backup-46"},"produto_id":{"S":"produto-mouse"},"quantidade":{"N":"2"},"nome_produto":{"S":"Mouse"},"preco":{"N":"80"},"atualizado_em":{"S":"2024-01-01T14:00:00Z"}}'
  aws dynamodb query --region us-east-1 --table-name carrinho-lab \
    --key-condition-expression "cliente_id = :c" \
    --expression-attribute-values '{":c":{"S":"cliente-backup-46"}}'
  ```
- **Esperado:** o `query` retorna `Count: 2` (anote como "verdade" a comparar).

### T4B.2 — BACKUP (gerar Recovery Point)
- **Prova:** um ponto de recuperação on-demand é criado.
- **Como:** AWS Backup → **Painel** → **Criar backup sob demanda** → DynamoDB, tabela `carrinho-lab`, **Criar backup agora**, cofre `db-lab-04-vault`, retenção 7 dias, role `AWSBackupDefaultServiceRole` → acompanhar em **Trabalhos → Trabalhos de backup**.
- **Esperado:** job **Concluído**; o cofre passa de 0 para 1 Recovery Point.

### T4B.3 — VALIDAR o backup
- **Prova:** o Recovery Point é íntegro e aponta para a tabela certa.
- **Como:** AWS Backup → **Cofres de backup** → `db-lab-04-vault` → **Pontos de recuperação**.
- **Esperado:** Status **Concluído**, origem `carrinho-lab`, tamanho > 0, expiração ~7 dias.

### T4B.4 — RESTORE (nova tabela)
- **Prova:** o restore cria **nova tabela** (nunca sobrescreve a origem).
- **Como:** no Recovery Point → **Restaurar** → nome **`carrinho-lab-restore`**, Sob demanda, role `AWSBackupDefaultServiceRole` → acompanhar em **Trabalhos → Trabalhos de restauração**.
- **Esperado:** job **Concluído**; `carrinho-lab-restore` **Ativa**; `carrinho-lab` intacta.

### T4B.5 — VALIDAR DADOS (fim do ciclo)
- **Prova:** os dados do momento do backup voltaram.
- **Como:**
  ```bash
  aws dynamodb query --region us-east-1 --table-name carrinho-lab-restore \
    --key-condition-expression "cliente_id = :c" \
    --expression-attribute-values '{":c":{"S":"cliente-backup-46"}}'
  ```
- **Esperado:** os **2 itens** conhecidos (Notebook qtd 1 preço 3500; Mouse qtd 2 preço 80), idênticos a T4B.1.

> **Limpeza (ciclo):** `aws dynamodb delete-table --region us-east-1 --table-name carrinho-lab-restore`

> **Conceito:** AWS Backup = Recovery Points discretos, cofre centralizado, retenção por política (governança). PITR = backup contínuo (RPO de segundos). Ambos criam nova tabela ao restaurar. RPO do AWS Backup ≈ frequência do plano.

---

## Observabilidade (EventBridge → CloudWatch Logs)

### T4E.1 — Eventos de Backup/Restore no Log Group
- **Prova:** o fluxo `AWS Backup → EventBridge → CloudWatch Logs` é auditável (mesmo padrão do RDS no Projeto 01).
- **Pré:** Log Group `/aws/events/db-lab-04-backup` e regra `db-lab-04-backup-eventos` criados (ver `IMPLANTACAO.md`, Fase 4).
- **Como:**
  1. Dispare um **backup on-demand** de `carrinho-lab` (repita T4B.2) e **anote o `BackupJobId`**.
  2. CloudWatch → **Logs → Insights** → grupo `/aws/events/db-lab-04-backup` → ajustar o intervalo (UTC) → rodar:
     ```text
     fields @timestamp, `detail-type` as tipo, detail.state as estado,
            detail.backupJobId, detail.restoreJobId, detail.resourceArn, detail.backupVaultName
     | sort @timestamp desc
     | limit 50
     ```
- **Esperado:** eventos `Backup Job State Change` com `estado` até **`COMPLETED`** e `detail.backupJobId` **igual** ao do Console — correlação que fecha a prova de ponta a ponta.

---

## Resumo dos testes

| ID | Teste | Comprova |
|---|---|---|
| T4G.0 | Réplica ativa | Existência da réplica |
| T4G.1 | Replicação origem → réplica | Multi-Region |
| T4G.2 | Replicação réplica → origem | Multi-Active |
| T4G.3 | `ReplicationLatency` | Replicação mensurável |
| T4B.0 | Estrutura de backup | Vault + Plan + seleção |
| T4B.1 | Dados conhecidos | Pré-condição do ciclo |
| T4B.2 | BACKUP | Recovery Point gerado |
| T4B.3 | VALIDAR backup | Recovery Point íntegro |
| T4B.4 | RESTORE | Nova tabela restaurada |
| T4B.5 | VALIDAR dados | Dados do backup voltaram |
| T4E.1 | Eventos no Log Group | Fluxo auditável |
