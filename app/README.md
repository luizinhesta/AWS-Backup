# app/ — Projeto 04 (reaproveitamento do Projeto 01)

> **Esta pasta é intencionalmente vazia de código.** Não há duplicação de código-fonte aqui.

## Fonte de verdade do código

O código-fonte da aplicação **AWS Database Lab Store** vive em um único lugar:

```text
aws-database-evolution-labs/01-rds-resilience/app/
```

O Projeto 04 (Global Tables + AWS Backup) **reutiliza a mesma aplicação** dos Projetos 01, 02 e 03. As regiões deste projeto são: origem **`us-east-1`** (Norte da Virgínia) e réplica **`us-east-2`** (Ohio). Isso atende ao Requisito 13.7 (reaproveitar o máximo, sem recriar aplicação, layout ou componentes) e ao Requisito 10.1 (o projeto reaproveita `dynamodb-lab.inhesta.net` e **não cria nova aplicação**).

## Por que não copiamos o código?

Copiar `app/` para dentro deste diretório criaria cópias do mesmo código, que precisariam ser mantidas em sincronia. Assim como nos Projetos 02 e 03, mantemos **uma única cópia** em `01-rds-resilience/app/` e documentamos apenas o **delta** do Projeto 04.

## O que muda no Projeto 04 (o delta)

O Projeto 04 tem **delta de infraestrutura, não de código de aplicação.** A aplicação continua exatamente igual à do Projeto 03: o carrinho já usa o DynamoDB (`CART_MODE=dynamodb`, tabela `carrinho-lab`) e produtos/clientes/pedidos seguem no relacional.

O que muda no Projeto 04 é **fora da aplicação**:

1. **DynamoDB Global Tables** — a tabela `carrinho-lab` ganha uma **réplica em `us-east-2`** (origem `us-east-1`), passando a operar multi-região e multi-active. Isso é **configuração de infraestrutura no Console**, feita na própria tabela; **nenhuma linha de código novo é necessária** para o carrinho continuar funcionando, porque o app já lê/grava em `carrinho-lab` na região configurada.
2. **AWS Backup** — Backup Vault, Backup Plan, Backup Rule, seleção de recursos, geração de Recovery Point e Restore. Também é **infraestrutura no Console**, sem impacto no código da aplicação.

### Configuração (`.env` na EC2)

Nada precisa mudar no `.env` para o carrinho continuar funcionando após ativar as Global Tables — o app continua apontando para `carrinho-lab` na região atual (ex.: `us-east-1`), e a AWS replica de forma transparente para `us-east-2`.

| Variável | Projeto 03 (DynamoDB single-region) | Projeto 04 (Global Tables) |
|---|---|---|
| `CART_MODE` | `dynamodb` | **`dynamodb`** (igual) |
| `DYNAMODB_TABLE` | `carrinho-lab` | **`carrinho-lab`** (igual — agora é uma tabela global) |
| `AWS_REGION` / `DYNAMODB_REGION` | região da tabela (ex.: `us-east-1`) | **igual** — o app segue usando a região local; a réplica em `us-east-2` é servida pela própria AWS |
| `LAB_PROJECT` | `Projeto 03 — AWS DynamoDB Recovery Lab` | `Projeto 04 — AWS Global Database & Backup Lab` |
| `LAB_ARCH_VERSION` | `Projeto 03 — Carrinho no DynamoDB (carrinho-lab)` | `Projeto 04 — DynamoDB Global Tables (us-east-1 ⇄ us-east-2) + AWS Backup` |

As variáveis de S3 e do relacional permanecem iguais — imagens no mesmo bucket e produtos/clientes/pedidos no mesmo banco relacional.

## Delta opcional (não essencial)

O essencial do Projeto 04 é **infraestrutura**. A aplicação **não precisa** de mudanças para se beneficiar das Global Tables: escrevendo na região local (`us-east-1`), a AWS replica automaticamente para `us-east-2` (e vice-versa).

Caso, para fins didáticos, a aplicação precisasse **escrever/ler explicitamente em `us-east-2`** (por exemplo, para demonstrar leitura de baixa latência a partir da outra região, ou um cenário de failover regional no lado do app), isso seria um **delta opcional de código**: criar um segundo cliente `boto3` apontando para `us-east-2` e escolher a região por configuração. **Não é necessário** para cumprir o Requisito 10 — a replicação bidirecional é comprovada testando escrita/leitura direto no Console/CLI de cada região (Tarefa 44), sem tocar no app.

Se esse delta opcional for implementado no futuro, ele deve ser feito **na fonte de verdade** `01-rds-resilience/app/cart.py` (ou em um pequeno módulo auxiliar), nunca com uma cópia dentro desta pasta.

## Como executar o Projeto 04

1. Reutilize o mesmo código de `01-rds-resilience/app/` na EC2 (a mesma pasta `/opt/aws-database-lab/` já implantada nos projetos anteriores).
2. Confirme que o Projeto 03 está funcional (carrinho no DynamoDB, `CART_MODE=dynamodb`).
3. Adicione a **réplica em `us-east-2`** à tabela `carrinho-lab` pelo Console (Tarefa 43) — sem mexer no código.
4. Configure o **AWS Backup** (Tarefas 45-47) — também sem mexer no código.
5. Opcional: atualize `LAB_PROJECT`/`LAB_ARCH_VERSION` no `.env` para refletir o Projeto 04 na página `/database-lab`.

Nenhum `template/`, `static/` ou lógica de rota precisa ser alterado ou copiado — o delta do Projeto 04 é de infraestrutura.
