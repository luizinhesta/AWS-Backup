# Arquitetura — Projeto 04 (Global Database & Backup Lab)

## ANTES vs DEPOIS

```text
ANTES
DynamoDB (carrinho-lab) — região única: us-east-1

DEPOIS
DynamoDB Global Tables (multi-active):
   us-east-1  ⇄  us-east-2
+ AWS Backup (Vault, Plan, Rule, Recovery Point, Restore)
```

## Replicação multi-região (bidirecional)

```text
escrita em us-east-1  →  replica em us-east-2
escrita em us-east-2  →  replica em us-east-1
Métrica de interesse: ReplicationLatency
```

## AWS Backup — ciclo completo

```text
Backup Vault → Backup Plan → Backup Rule → Seleção de recursos
      ↓
Recovery Point
      ↓
Restore
      ↓
Validar dados restaurados
```

## EventBridge + AWS Backup

```text
AWS Backup (Backup/Restore Job State Change)
      ↓
EventBridge
      ↓
CloudWatch Logs (/aws/events/db-lab-04-backup)
```

## Estado final da série e limpeza (Tarefa 48)

Este é o **último** projeto da série. Depois de comprovar Global Tables + AWS Backup, a arquitetura evoluída deixa de ser necessária e todos os recursos dos **4 projetos** são removidos para **zerar o custo** (Req. 12.2/12.3):

```text
DEPOIS (arquitetura evoluída)                  →  LIMPEZA FINAL (custo zerado)
DynamoDB Global Tables (us-east-1 ⇄ us-east-2)    → remover réplica us-east-2 → excluir carrinho-lab
AWS Backup (Vault/Plan/Rule/Recovery Points)      → excluir Recovery Points → excluir Vault
CloudWatch/EventBridge (db-lab-01/02/03/04)       → excluir dashboards/alarmes/regras/Log Groups
Aurora / RDS (+ Multi-AZ/Read Replica/Clone)      → excluir instâncias + snapshots (se existirem)
S3 (imagens) / EC2 / Elastic IP / Route 53        → esvaziar+excluir S3, terminar EC2, liberar EIP, remover DNS
Security Groups / IAM Roles                        → remover SGs; (opcional) remover Roles
```

Passo a passo (ordem por dependência + comandos) em `IMPLANTACAO.md` → seção **"Limpeza final (zerar custo — Tarefa 48)"**.
