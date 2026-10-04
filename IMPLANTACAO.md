# Implantação — Projeto 04 (Global Database & Backup)

> Guia **passo a passo** pelo **Console AWS (Português-Brasil)**, com indicação exata de **onde clicar**, **o que preencher** e **o que esperar** em cada tela.
>
> **Delta do Projeto 04:** evoluir a tabela `carrinho-lab` para **DynamoDB Global Tables** e adicionar **AWS Backup** com observabilidade via **EventBridge → CloudWatch Logs**. A aplicação, a EC2 e o DNS (`dynamodb-lab.inhesta.net`) **não mudam** — o delta é só de infraestrutura.

## Regiões deste guia

| Papel | Região | Onde aparece |
|---|---|---|
| **Origem** (tudo é feito aqui) | **`us-east-1` — Leste dos EUA (Norte da Virgínia)** | Global Tables (origem), AWS Backup, EventBridge, CloudWatch |
| **Réplica** (2ª região da Global Table) | **`us-east-2` — Leste dos EUA (Ohio)** | Somente a criação da réplica e a leitura cruzada |

> **Como trocar de região no Console:** canto **superior direito** → clique no nome da região atual (ex.: "Estados Unidos (Norte da Virgínia)") → escolha a região desejada na lista. **Sempre confira a região** antes de criar qualquer recurso — recurso criado na região errada não aparece na região certa.
>
> Se preferir outra região para a réplica (ex.: `sa-east-1`), troque `us-east-2` por ela em todos os comandos e telas abaixo.

## Onde rodar os comandos (AWS CLI)

Os blocos de comando deste guia são para o **AWS CloudShell** (ícone de terminal `>_` no topo do Console, ao lado do sino de notificações):

- O CloudShell já vem com a **AWS CLI instalada e autenticada** com o seu usuário do Console — não precisa de `aws configure` nem chaves.
- É um shell **Linux (bash)**, então aspas simples `'{...}'`, `sleep 5` e a quebra de linha com `\` funcionam exatamente como escritos.
- Cada comando já traz `--region` explícito, então funciona independentemente da região em que o CloudShell abriu.
- Alternativa: rodar na **EC2 via SSH** (também Linux/bash). Em **PowerShell no Windows** a sintaxe de aspas muda — prefira o CloudShell.

> **⚠️ ATENÇÃO DE CUSTO**
> - **Global Tables:** a réplica em `us-east-2` cobra **armazenamento na 2ª região** + **gravações replicadas (rWCU)**. Começa no instante em que a réplica é criada.
> - **AWS Backup:** cobra **armazenamento dos Recovery Points** (GB-mês) + cada **restore**. Começa ao gerar o 1º Recovery Point.
> - Volumes do carrinho são de poucos KB, então os valores são baixos — mas **existem**. **Remova** a réplica e os Recovery Points ao concluir os testes (seção **Limpeza final**).

---

## Fase 1 — DynamoDB Global Tables (Tarefas 43-44)

**Objetivo:** transformar `carrinho-lab` (hoje só em `us-east-1`) numa tabela global, adicionando uma réplica em `us-east-2`, e comprovar a replicação nos dois sentidos.

### Passo 1.1 — Confirmar a tabela de origem

1. No **topo direito**, confirme a região **Estados Unidos (Norte da Virgínia) us-east-1**.
2. Barra de busca (topo) → digite **DynamoDB** → clique no serviço **DynamoDB**.
3. No menu lateral esquerdo, clique em **Tabelas**.
4. Na lista, clique na tabela **`carrinho-lab`**.
5. Confira no **Resumo** que o **Status** é **Ativo** e o **Modo de capacidade** é **Sob demanda**.

> **O que observar:** a tabela já existe desde o Projeto 03. Se ela não aparecer, você provavelmente está na região errada — volte ao passo 1.

### Passo 1.2 — Criar a réplica em `us-east-2`

1. Ainda dentro de **`carrinho-lab`**, clique na aba **Tabelas globais** (fica na faixa de abas: Configurações · Índices · Monitorar · **Tabelas globais** · Backups · ...).
2. Clique no botão **Criar réplica** (canto direito do bloco **Réplicas**).
3. Na tela **Criar réplica → Configurações de replicação**:
   - **Região atual:** mostra `us-east-1` (é a origem — correto).
   - **Perfil do IAM:** `AWSServiceRoleForDynamoDBReplication` já vem preenchido — **não altere** (é a role vinculada ao serviço, criada automaticamente).
   - **Consistência:** deixe **Consistência eventual** (opção padrão, à esquerda). *(Consistência forte global é mais nova, mais cara e desnecessária no lab.)*
   - **Regiões de replicação disponíveis:** no menu suspenso, selecione **Estados Unidos (Ohio) us-east-2**.
   - Note o aviso: **"o DynamoDB Streams será ativado automaticamente para imagens novas e antigas"** — isso é obrigatório para Global Tables e já vem resolvido. Nada a fazer.
4. Clique em **Criar** (botão amarelo, canto inferior direito).

![Descrição da imagem](<imagens/imagem%20(4).png>)
![Descrição da imagem](<imagens/imagem%20(25).png>)

**O que acontece depois:**
- Você volta para a aba **Tabelas globais**. No bloco **Réplicas (2)** aparecem duas linhas:
  - `us-east-1` — **Status: Ativo** (Região atual).
  - `us-east-2` — **Status: Criando** (fica alguns minutos nesse estado).
- Clique no ícone de **atualizar** (🔄, acima do bloco Réplicas) de tempos em tempos até `us-east-2` virar **Ativo**.
- Cada réplica mostra também o **Endpoint** regional (`dynamodb.us-east-2.amazonaws.com`), o **Tipo** (Réplica / Região atual) e o **Modo de capacidade** (Sob demanda).

> **Não avance** para o Passo 1.3 enquanto `us-east-2` não estiver **Ativo** — o `get-item` cruzado falharia porque a réplica ainda não existe.

### Passo 1.3 — Validar por operação real (não basta o status "Ativo")

Abra o **CloudShell** (`>_` no topo) e rode, na ordem:

```bash
# 1) Gravar um item na origem us-east-1
aws dynamodb put-item --region us-east-1 --table-name carrinho-lab \
  --item '{"cliente_id":{"S":"cliente-teste-43"},"produto_id":{"S":"produto-global"},"quantidade":{"N":"1"},"nome_produto":{"S":"Teste Global Tables"},"preco":{"N":"10"},"atualizado_em":{"S":"2024-01-01T12:00:00Z"}}'

# 2) Aguardar a replicação (consistência eventual entre regiões, ~1s)
sleep 5

# 3) Ler o MESMO item na réplica us-east-2 (deve retornar o Item)
aws dynamodb get-item --region us-east-2 --table-name carrinho-lab \
  --key '{"cliente_id":{"S":"cliente-teste-43"},"produto_id":{"S":"produto-global"}}'

# 4) Limpar o item de teste (a exclusão também replica)
aws dynamodb delete-item --region us-east-1 --table-name carrinho-lab \
  --key '{"cliente_id":{"S":"cliente-teste-43"},"produto_id":{"S":"produto-global"}}'

```
![Descrição da imagem](<imagens/imagem%20(2).png>)
![Descrição da imagem](<imagens/imagem%20(3).png>)

**O que esperar:**
- O `put-item` não imprime nada (sucesso silencioso).
- O `get-item` em `us-east-2` retorna um JSON com o campo **`Item`** contendo `cliente-teste-43`. **Isso é a prova real da replicação.**
- Se o `Item` não vier na primeira tentativa, aguarde mais alguns segundos e repita só o `get-item` (replicação é eventual).

### Passo 1.4 — Teste bidirecional + métrica `ReplicationLatency`

**Sentido A — `us-east-1 → us-east-2` (Multi-Region):**
```bash
aws dynamodb put-item --region us-east-1 --table-name carrinho-lab \
  --item '{"cliente_id":{"S":"cliente-teste-44"},"produto_id":{"S":"produto-e1-e2"},"quantidade":{"N":"2"},"nome_produto":{"S":"Gravado em us-east-1"},"preco":{"N":"99"},"atualizado_em":{"S":"2024-01-01T13:00:00Z"}}'
sleep 5
aws dynamodb get-item --region us-east-2 --table-name carrinho-lab \
  --key '{"cliente_id":{"S":"cliente-teste-44"},"produto_id":{"S":"produto-e1-e2"}}'
```

**Sentido B — `us-east-2 → us-east-1` (Multi-Active — prova que a 2ª região também ESCREVE):**
```bash
aws dynamodb put-item --region us-east-2 --table-name carrinho-lab \
  --item '{"cliente_id":{"S":"cliente-teste-44"},"produto_id":{"S":"produto-e2-e1"},"quantidade":{"N":"3"},"nome_produto":{"S":"Gravado em us-east-2"},"preco":{"N":"150"},"atualizado_em":{"S":"2024-01-01T13:05:00Z"}}'
sleep 5
aws dynamodb get-item --region us-east-1 --table-name carrinho-lab \
  --key '{"cliente_id":{"S":"cliente-teste-44"},"produto_id":{"S":"produto-e2-e1"}}'
```

**Ver a métrica `ReplicationLatency` no Console:**
1. Região **us-east-1** → busca → **CloudWatch**.
2. Menu lateral → **Métricas** → **Todas as métricas**.
3. Clique no namespace **`AWS/DynamoDB`**.
4. Escolha o grupo de dimensões que contém **`ReceivingRegion`** (ex.: "Table Metrics by Receiving Region").
5. Marque a métrica **`ReplicationLatency`** com `TableName = carrinho-lab` e `ReceivingRegion = us-east-2` (marque também `us-east-1` para ver o outro sentido).
6. Na aba **Gráfico**, ajuste **Período = 1 minuto** e **Estatística = Média** (e **Máximo**).

**Limpeza dos itens de teste:**
```bash
aws dynamodb delete-item --region us-east-1 --table-name carrinho-lab \
  --key '{"cliente_id":{"S":"cliente-teste-44"},"produto_id":{"S":"produto-e1-e2"}}'
aws dynamodb delete-item --region us-east-1 --table-name carrinho-lab \
  --key '{"cliente_id":{"S":"cliente-teste-44"},"produto_id":{"S":"produto-e2-e1"}}'
```

**O que esperar:** valores de `ReplicationLatency` tipicamente entre **centenas de ms e ~1s** logo após os `put-item`. A existência de pontos no gráfico confirma replicação ativa e mensurável.

> **Itens relacionados a esta fase:**
> - **Conceito SAA:** Global Tables = **Multi-Region + Multi-Active + replicação assíncrona** (consistência eventual; conflito do mesmo item resolvido por *last writer wins* pelo timestamp). Difere de **Multi-AZ** (disponibilidade dentro da região) e **Read Replica** (escala de leitura dentro da região).
> - **DynamoDB Streams:** habilitado automaticamente ao criar a 1ª réplica (visão "Nova e antiga imagem") — é a infraestrutura que carrega a replicação. Você vê isso na aba **Exportações e streams**.
> - **(Opcional) IAM para a aplicação:** se o app (boto3 na EC2) for acessar `us-east-2`, edite a política em linha `politica-carrinho-dynamodb-lab` da role `ec2-imagens-s3-role` (IAM → Funções) e **acrescente** o ARN `arn:aws:dynamodb:us-east-2:<ACCOUNT_ID>:table/carrinho-lab` ao `Resource`. Para teste via CLI/CloudShell com seu usuário admin, **não é necessário**.

![Descrição da imagem](<imagens/imagem%20(13).png>)

---

## Fase 2 — AWS Backup: criar a estrutura (Tarefa 45)

**Objetivo:** montar o Cofre + Plano + Regra + seleção que vão proteger `carrinho-lab`. Toda a Fase 2 é em **`us-east-1`** (AWS Backup para DynamoDB é **regional** — protege a réplica local). Criar Vault/Plan/Rule **não gera custo**; só os Recovery Points cobram.

![Descrição da imagem](<imagens/imagem%20(5).png>)

### Passo 2.1 — Criar o Cofre de backup (Backup Vault)

![Descrição da imagem](<imagens/imagem%20(6).png>)

1. Confirme a região **us-east-1** no topo direito.
2. Busca → **AWS Backup** → abra o serviço.
3. Menu lateral → **Cofres de backup** (Backup vaults).
4. Clique em **Criar cofre de backup** (Create backup vault).
5. Preencha:
   - **Nome do cofre de backup:** **`db-lab-04-vault`**.
   - **Chave de criptografia do KMS:** deixe o **padrão** (`aws/backup`).
   - **Tags** (opcional): `projeto = db-lab-04`.
6. Clique em **Criar cofre de backup**.

**O que esperar:** o cofre **`db-lab-04-vault`** aparece na lista com **Pontos de recuperação = 0**. Criar o cofre não cobra nada.

![Descrição da imagem](<imagens/imagem%20(19).png>)

### Passo 2.2 — Criar o Plano de backup (Backup Plan) + Regra (Rule)

1. Menu lateral → **Planos de backup** (Backup plans).
2. Clique em **Criar plano de backup**.
3. Escolha **Começar do zero** / **Criar novo plano** (Build a new plan).
4. **Nome do plano de backup:** **`db-lab-04-plano`**.
5. Em **Configuração da regra de backup** (Backup rule), preencha:
   - **Nome da regra:** **`db-lab-04-regra-diaria`**.
   - **Cofre de backup:** selecione **`db-lab-04-vault`**.
   - **Frequência do backup:** **Diariamente** (Daily).
   - **Janela de backup:** deixe **Usar padrões** (o disparo real deste lab será on-demand na Fase 3).
   - **Ciclo de vida / Retenção:** **Expirar após 7 dias** (Retention = 7 days). *(Retenção curta de propósito, para limitar custo.)*
   - **Mover para armazenamento frio:** **não** habilitar.
   - **Copiar para outra região/cofre:** **não** habilitar (evita custo de cópia entre regiões).
6. Clique em **Criar plano** (Create plan).

**O que esperar:** o plano **`db-lab-04-plano`** aparece em **Planos de backup** contendo a regra `db-lab-04-regra-diaria`, com cofre `db-lab-04-vault` e retenção de 7 dias. Ainda sem custo.

![Descrição da imagem](<imagens/imagem%20(10).png>)
![Descrição da imagem](<imagens/imagem%20(23).png>)

### Passo 2.3 — Atribuir recursos (selecionar `carrinho-lab`)

> Sem este passo o plano existe mas **não faz backup de nada**.

1. Em **Planos de backup**, clique em **`db-lab-04-plano`** para abri-lo.
2. Role até a seção **Recursos atribuídos** (Assigned resources) → clique em **Atribuir recursos** (Assign resources).
3. Preencha:
   - **Nome da atribuição de recurso:** **`db-lab-04-carrinho`**.
   - **Função do IAM:** selecione **Função padrão** → **`AWSBackupDefaultServiceRole`**. *(Na primeira vez, a AWS cria essa role automaticamente, já com as políticas gerenciadas de backup e restore — menor privilégio.)*
   - **Atribuir recursos:** escolha **Incluir tipos de recursos específicos** (Include specific resource types) → tipo **DynamoDB** → selecione a tabela **`carrinho-lab`** (o Console lista as tabelas de `us-east-1`).
4. Clique em **Atribuir recursos**.

**O que esperar:** dentro de `db-lab-04-plano`, a seção **Recursos atribuídos** lista `db-lab-04-carrinho` apontando para `carrinho-lab`, com a role `AWSBackupDefaultServiceRole`. Ainda **não há Recovery Point** — isso é esperado nesta fase.

![Descrição da imagem](<imagens/imagem%20(8).png>)

### Passo 2.4 — Conferência (opcional, via CloudShell)

```bash
# Deve listar o plano db-lab-04-plano
aws backup list-backup-plans --region us-east-1

# Pegue o BackupPlanId do comando acima e confirme a seleção de recursos
aws backup list-backup-selections --region us-east-1 --backup-plan-id <BACKUP_PLAN_ID>
```

> **Itens relacionados a esta fase:**
> - **AWS Backup é regional para DynamoDB:** ele faz backup da **réplica da região onde o Backup é configurado** (`us-east-1`), não da Global Table inteira. Para proteger também `us-east-2`, repetiria a configuração lá (não é necessário no lab).
> - **IAM Role de serviço:** quem executa backup/restore é o **serviço** `backup.amazonaws.com` assumindo a `AWSBackupDefaultServiceRole` — não um usuário. Você vê a trust policy em IAM → Funções → `AWSBackupDefaultServiceRole`.
> - **Vault vs. Plan vs. Rule vs. Seleção:** o **Vault** guarda; o **Plan** agrupa regras; a **Rule** define *para onde, com que frequência e por quanto tempo*; a **Seleção** define *quais recursos*.

---

## Fase 3 — Ciclo BACKUP → VALIDAR → RESTORE → VALIDAR DADOS (Tarefa 46)

**Objetivo:** provar que o backup é **utilizável** — gerar um Recovery Point, restaurar numa nova tabela e conferir os dados. Tudo em **`us-east-1`**. O teste **não termina ao criar o backup**.

### Passo 3.1 — Preparar dados conhecidos (pré-condição)

No CloudShell:
```bash
aws dynamodb put-item --region us-east-1 --table-name carrinho-lab \
  --item '{"cliente_id":{"S":"cliente-backup-46"},"produto_id":{"S":"produto-notebook"},"quantidade":{"N":"1"},"nome_produto":{"S":"Notebook"},"preco":{"N":"3500"},"atualizado_em":{"S":"2024-01-01T14:00:00Z"}}'
aws dynamodb put-item --region us-east-1 --table-name carrinho-lab \
  --item '{"cliente_id":{"S":"cliente-backup-46"},"produto_id":{"S":"produto-mouse"},"quantidade":{"N":"2"},"nome_produto":{"S":"Mouse"},"preco":{"N":"80"},"atualizado_em":{"S":"2024-01-01T14:00:00Z"}}'

# Conferir o estado conhecido antes do backup (deve dar Count: 2)
aws dynamodb query --region us-east-1 --table-name carrinho-lab \
  --key-condition-expression "cliente_id = :c" \
  --expression-attribute-values '{":c":{"S":"cliente-backup-46"}}'
```

![Descrição da imagem](<imagens/imagem%20(9).png>)
![Descrição da imagem](<imagens/imagem%20(17).png>)

> **Por quê:** para validar o restore depois, precisamos de itens conhecidos e estáveis na tabela no momento do backup. Anote/print esse `Count: 2` — é a "verdade" que vamos comparar.

### Passo 3.2 — BACKUP: gerar um Recovery Point on-demand

1. Região **us-east-1** → **AWS Backup** → menu lateral **Painel** (Dashboard).
2. Clique em **Criar backup sob demanda** (Create on-demand backup).
3. Preencha:
   - **Tipo de recurso:** **DynamoDB**.
   - **Nome da tabela:** **`carrinho-lab`**.
   - **Criar backup agora** (para disparo imediato, em vez de agendar).
   - **Retenção:** **7 dias**.
   - **Cofre de backup:** **`db-lab-04-vault`**.
   - **Função do IAM:** **Função padrão** → **`AWSBackupDefaultServiceRole`**.
4. Clique em **Criar backup sob demanda**.
5. Acompanhe em **AWS Backup → Trabalhos → Trabalhos de backup** (Jobs → Backup jobs): o job vai de **Em execução** (Running) a **Concluído** (Completed) — de segundos a poucos minutos para DynamoDB.

![Descrição da imagem](<imagens/imagem%20(11).png>)

> **Alternativa CLI:**
> ```bash
> aws backup start-backup-job --region us-east-1 \
>   --backup-vault-name db-lab-04-vault \
>   --resource-arn arn:aws:dynamodb:us-east-1:<ACCOUNT_ID>:table/carrinho-lab \
>   --iam-role-arn arn:aws:iam::<ACCOUNT_ID>:role/service-role/AWSBackupDefaultServiceRole \
>   --lifecycle DeleteAfterDays=7
> ```

### Passo 3.3 — VALIDAR o backup (conferir o Recovery Point)

1. **AWS Backup** → **Cofres de backup** → abra **`db-lab-04-vault`**.
2. Na lista **Pontos de recuperação** (Recovery points), confira:
   - **Status:** **Concluído** (Completed).
   - **Recurso de origem:** `carrinho-lab` (tipo DynamoDB).
   - **Tamanho do backup:** poucos KB.
   - **Data de expiração:** ~7 dias após a criação.

**O que esperar:** o cofre passa de **0** para **1 ponto de recuperação**.

### Passo 3.4 — RESTORE para uma NOVA tabela

> O restore do DynamoDB **sempre cria uma nova tabela** — nunca sobrescreve a origem. Por isso usamos um nome diferente.

1. Ainda em **`db-lab-04-vault`** → **Pontos de recuperação** → marque o Recovery Point do Passo 3.3 → clique em **Restaurar** (Restore).
2. Na tela de restauração do DynamoDB:
   - **Nome da nova tabela:** **`carrinho-lab-restore`**.
   - **Configurações / capacidade:** mantenha o **padrão** (Sob demanda). Não precisa recriar Global Table nem PITR.
   - **Função do IAM:** **Função padrão** → **`AWSBackupDefaultServiceRole`**.
3. Clique em **Restaurar backup** (Restore backup).
4. Acompanhe em **AWS Backup → Trabalhos → Trabalhos de restauração** até **Concluído**.
5. Vá em **DynamoDB → Tabelas** e confirme que **`carrinho-lab-restore`** está **Ativa**.

### Passo 3.5 — VALIDAR DADOS (comparar com os itens conhecidos)

```bash
aws dynamodb query --region us-east-1 --table-name carrinho-lab-restore \
  --key-condition-expression "cliente_id = :c" \
  --expression-attribute-values '{":c":{"S":"cliente-backup-46"}}'
```

**O que esperar:** retorna os **2 itens** (`Notebook` qtd 1 preço 3500; `Mouse` qtd 2 preço 80), idênticos ao Passo 3.1. A origem `carrinho-lab` fica **intacta**. **Só agora** o ciclo está completo.

### Passo 3.6 — Limpeza do ciclo (controle de custo)

```bash
# Excluir a tabela restaurada (não é mais necessária)
aws dynamodb delete-table --region us-east-1 --table-name carrinho-lab-restore
```
Ou pelo Console: **DynamoDB → Tabelas → `carrinho-lab-restore` → Excluir**.

> Mantenha o Recovery Point para a Fase 4 (ou remova na Limpeza final).

> **Itens relacionados a esta fase:**
> - **AWS Backup vs. PITR:** AWS Backup gera **Recovery Points discretos** num cofre centralizado e auditável (governança, retenção por política). PITR (Projeto 03) é **backup contínuo** com RPO de segundos, dentro da região. Ambos criam **nova tabela** ao restaurar.
> - **RTO/RPO:** o **RPO** do AWS Backup é limitado pela **frequência** do plano (diário ⇒ até ~24h); o **RTO** depende do tempo de restore + redirecionar a aplicação para a nova tabela.
> - **Um backup só é confiável depois de restaurado** — por isso o ciclo termina em **VALIDAR DADOS**, não em "criar o backup".

---
![Descrição da imagem](<imagens/imagem%20(26).png>)

## Fase 4 — Observabilidade: EventBridge → CloudWatch Logs (Tarefa 47)

**Objetivo:** registrar automaticamente cada mudança de estado dos jobs de backup/restore num Log Group pesquisável. Tudo em **`us-east-1`**.

### Passo 4.1 — Criar o Log Group de destino

1. Região **us-east-1** → busca → **CloudWatch**.
2. Menu lateral → **Logs** → **Grupos de logs** (Log groups).
3. Clique em **Criar grupo de logs** (Create log group).
4. Preencha:
   - **Nome do grupo de logs:** **`/aws/events/db-lab-04-backup`** (exatamente assim — `/aws/events/` é a convenção da série).
   - **Retenção:** **30 dias** (evita acúmulo de logs).
5. Clique em **Criar** (Create).

**O que esperar:** o grupo `/aws/events/db-lab-04-backup` aparece na lista, ainda **sem fluxos de log** (eles surgem quando o 1º evento chegar, no Passo 4.3).

### Passo 4.2 — Criar a regra do EventBridge

1. Região **us-east-1** → busca → **Amazon EventBridge**.
2. Menu lateral → **Regras** (Rules). Confirme o **Barramento de eventos** = **`default`**.
3. Clique em **Criar regra** (Create rule).
4. **Etapa 1 — Detalhes da regra:**
   - **Nome:** **`db-lab-04-backup-eventos`**.
   - **Descrição:** `Encaminha eventos de Backup/Restore Job State Change do AWS Backup para o Log Group /aws/events/db-lab-04-backup`.
   - **Barramento:** **`default`**.
   - **Tipo de regra:** **Regra com um padrão de evento** → **Próximo**.
5. **Etapa 2 — Padrão de evento:**
   - **Origem do evento:** **Eventos de serviços da AWS**.
   - **Método de criação:** escolha **Padrão personalizado (editor JSON)** e cole:
     ```json
     {
       "source": ["aws.backup"],
       "detail-type": ["Backup Job State Change", "Restore Job State Change"]
     }
     ```
   - **Próximo**.
6. **Etapa 3 — Selecionar alvos:**
   - **Tipo de alvo:** **Serviço da AWS**.
   - **Selecionar um alvo:** **Grupo de logs do CloudWatch** (CloudWatch log group).
   - **Grupo de logs:** selecione **`/aws/events/db-lab-04-backup`**.
   - **Próximo** nas etapas seguintes (tags opcionais) → **Criar regra**.

**O que esperar:** a regra `db-lab-04-backup-eventos` aparece **Habilitada** em EventBridge → Regras, com o padrão `aws.backup` e o alvo apontando para o Log Group. O EventBridge cria sozinho a permissão de escrita no Log Group.

### Passo 4.3 — Gerar eventos e localizá-los (comprovação)

1. Dispare um **backup on-demand** de `carrinho-lab` (repita o Passo 3.2) e **anote o `BackupJobId`** (visto em **Trabalhos → Trabalhos de backup**).
2. Aguarde o job chegar a **Concluído**.
3. CloudWatch → **Logs** → **Insights** (Logs Insights).
4. Em **Selecionar grupos de logs**, escolha **`/aws/events/db-lab-04-backup`**.
5. Ajuste o **intervalo de tempo** para cobrir o horário do backup (eventos são gravados em **UTC**).
6. Cole a query e clique em **Executar consulta** (Run query):
   ```text
   fields @timestamp, `detail-type` as tipo, detail.state as estado,
          detail.backupJobId, detail.restoreJobId, detail.resourceArn, detail.backupVaultName
   | sort @timestamp desc
   | limit 50
   ```

**O que esperar:** linhas com `detail-type = "Backup Job State Change"` e `estado` evoluindo até **`COMPLETED`**, com `detail.backupJobId` **igual** ao do Console e `detail.resourceArn` da tabela `carrinho-lab`. Essa correlação comprova o fluxo `AWS Backup → EventBridge → CloudWatch Logs`.

> **Itens relacionados a esta fase:**
> - **Padrão da série:** este é o mesmo fluxo `serviço → EventBridge → CloudWatch Logs` usado no Projeto 01 (RDS, failover) e no Projeto 03 (DynamoDB). Muda só o `source` (`aws.backup`) e o `detail-type`.
> - **Filtrar só estados finais (opcional):** acrescente ao JSON `"detail": {"state": ["COMPLETED","FAILED","ABORTED"]}` para registrar apenas conclusões/falhas.
> - **Campos úteis do evento:** `Backup Job State Change` traz `backupJobId`, `state`, `resourceArn`, `backupVaultName`; `Restore Job State Change` traz `restoreJobId`, `state`, `createdResourceArn` (a tabela restaurada).

---

## Limpeza final (zerar custo — Tarefa 48)

Siga **nesta ordem** (respeita dependências). Tudo é iniciado a partir de **`us-east-1`**.

### 1. Remover a réplica `us-east-2` da Global Table
- **Console:** DynamoDB (`us-east-1`) → **Tabelas** → `carrinho-lab` → aba **Tabelas globais** → marque a linha **`us-east-2`** → **Excluir réplica**.
- **CLI:**
  ```bash
  aws dynamodb update-table --region us-east-1 --table-name carrinho-lab \
    --replica-updates '[{"Delete":{"RegionName":"us-east-2"}}]'
  ```
- **Efeito:** fim do custo de 2ª região; o dado permanece em `us-east-1`.

### 2. Excluir tabelas de teste
- **Console:** DynamoDB (`us-east-1`) → **Tabelas** → excluir `carrinho-lab-restore` (e `carrinho-lab-restaurada`, se existir do Projeto 03).

### 3. Esvaziar e excluir o AWS Backup
> Um Vault só é excluído **vazio** — exclua os Recovery Points primeiro.
1. **AWS Backup → Cofres de backup → `db-lab-04-vault` → Pontos de recuperação** → selecionar cada ponto → **Excluir**.
2. **AWS Backup → Planos de backup → `db-lab-04-plano`** → remover a atribuição `db-lab-04-carrinho` → **Excluir plano**.
3. **AWS Backup → Cofres de backup → `db-lab-04-vault` → Excluir** (agora vazio).

### 4. PITR e tabela `carrinho-lab`
- **Desabilitar PITR:** DynamoDB → `carrinho-lab` → aba **Backups** → **Recuperação para um ponto no tempo** → **Desativar**.
- **Excluir a tabela** (se não for mais estudar o carrinho): DynamoDB → **Tabelas** → `carrinho-lab` → **Excluir**.

### 5. Observabilidade
- **EventBridge → Regras:** excluir `db-lab-04-backup-eventos`.
- **CloudWatch → Logs → Grupos de logs:** excluir `/aws/events/db-lab-04-backup`.
- Remova também os recursos equivalentes dos Projetos 01-03, se ainda existirem.

> **Limpeza completa de toda a série (Tarefa 48):** além dos recursos do Projeto 04 acima, encerre também os recursos dos Projetos 01-03 que ainda existirem — Aurora, RDS (Multi-AZ/Read Replica) e snapshots, bucket S3 de imagens, instância EC2 e Elastic IP, registros Route 53, Security Groups e, se quiser, as IAM Roles. **Ordem geral:** primeiro o que cobra por hora ligado (RDS/Aurora), depois armazenamento (snapshots, Recovery Points, S3) e por fim rede (Elastic IP ocioso, Route 53). Ao final, confira em **Faturamento (Billing) / Cost Explorer** que não há mais cobrança recorrente.
