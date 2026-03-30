# Guia de Integração de Dados — Protocolo Obrigatório para IA

> **REGRA FUNDAMENTAL:** Este guia é um protocolo de alinhamento com o usuário. A IA NÃO PODE escrever código de integração antes de completar TODAS as fases de alinhamento e receber confirmação explícita no checkpoint final. A IA é a executora, o usuário é o arquiteto da decisão.

---

## Índice

1. [Fase 1 — Discovery](#fase-1--discovery)
2. [Fase 2 — Decisão de Arquitetura](#fase-2--decisão-de-arquitetura)
3. [Fase 3 — Alinhamento Técnico Detalhado](#fase-3--alinhamento-técnico-detalhado)
4. [Fase 4 — Checkpoint de Consenso](#fase-4--checkpoint-de-consenso)
5. [Fase 5 — Implementação](#fase-5--implementação)
6. [Boas Práticas Obrigatórias](#boas-práticas-obrigatórias)
7. [Anti-Patterns (Proibições)](#anti-patterns-proibições)
8. [Referência SDK](#referência-sdk)

---

## Fase 1 — Discovery

```
┌─────────────────────────────────────────────────┐
│           PERGUNTAS OBRIGATÓRIAS                │
│         (todas devem ser respondidas)            │
└─────────────────────────────────────────────────┘
```

A IA DEVE fazer TODAS estas perguntas antes de avançar:

### 1.1 Sobre a Fonte de Dados
- [ ] **Qual é o sistema/fonte?** (ERP, CRM, banco externo, API, planilha CSV?)
- [ ] **Qual o tipo de acesso?** (banco de dados direto via JDBC? API REST? arquivo exportado?)
- [ ] **O sistema fonte é acessível pela internet ou está on-premise?** (se on-premise, vai precisar de Cloudflare Tunnel)
- [ ] **Você tem credenciais de leitura disponíveis?** (usuário read-only no banco? API key? token?)
- [ ] **Existe documentação da estrutura de dados?** (dicionário de dados, docs de API, schema do banco?)

### 1.2 Sobre o Volume e Uso
- [ ] **Qual o volume estimado de dados?** (centenas, milhares, milhões de registros?)
- [ ] **Quantos registros novos/alterados por dia?**
- [ ] **Quantos usuários vão consumir esses dados nos dashboards?**
- [ ] **Os dados são usados para análise/BI ou para operação transacional?**

### 1.3 Sobre a Necessidade de Atualização
- [ ] **Com que frequência os dados precisam estar atualizados?** (real-time, a cada hora, diário, semanal?)
- [ ] **Existe tolerância a delay?** (ex: "dados do dia anterior está ok" vs "preciso ver o pedido que acabou de entrar")
- [ ] **Existe janela de baixo uso do sistema fonte?** (madrugada, fim de semana? — para full syncs pesados)

### 1.4 Sobre Transformações
- [ ] **Os dados vêm prontos ou precisam de tratamento?** (joins, cálculos, limpeza, normalização?)
- [ ] **Existem regras de negócio que precisam ser aplicadas nos dados?** (classificações, agrupamentos, conversões?)

### 1.5 Sobre Estado Atual do Projeto
- [ ] **O projeto já possui tabelas com dados sample/fictícios?** (cenário muito comum: o sistema já foi construído com dados de exemplo e agora quer conectar ao ERP real)
- [ ] **Se sim: quais tabelas já existem e qual a estrutura delas?** (listar tabelas, colunas, tipos)
- [ ] **Podemos deletar todos os dados sample dessas tabelas para substituir pela integração real?** (perguntar explicitamente, ex: "Posso limpar os dados fictícios das tabelas CLIENTES, PEDIDOS, PRODUTOS e substituir pela importação automatizada do ERP?")
- [ ] **Existe algum dado que NÃO deve ser deletado?** (ex: configurações, tabelas de apoio que foram preenchidas manualmente)

> **PROIBIDO:** Avançar para Fase 2 sem resposta para TODAS as perguntas acima. Se o usuário não souber alguma, a IA deve ajudá-lo a descobrir ou registrar como pendência a ser investigada.

---

## Fase 2 — Decisão de Arquitetura

```
┌──────────────────────────────────────────────────────────────────────┐
│                    ÁRVORE DE DECISÃO                                │
│                                                                      │
│  ┌──────────┐     ┌───────────────┐     ┌─────────────────────┐     │
│  │  Fonte   │────▶│  Tipo de      │────▶│  Modelo escolhido   │     │
│  │  dados   │     │  acesso       │     │  pelo USUÁRIO       │     │
│  └──────────┘     └───────────────┘     └─────────────────────┘     │
│                                                                      │
│  A IA SUGERE, mas o USUÁRIO DECIDE                                  │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.1 Três Modelos de Consumo

A IA DEVE apresentar os 3 modelos ao usuário com prós e contras:

#### Modelo A — Online / Tempo Real
```
Usuário acessa dashboard → SF consulta fonte externa na hora → Dados frescos
```

| Prós | Contras |
|------|---------|
| Dados sempre atualizados | Cada acesso gera carga no ERP/sistema fonte |
| Sem necessidade de rotina de importação | Latência maior (depende da fonte) |
| Sem armazenamento duplicado | Se a fonte cair, o dashboard fica fora |
| Mais simples de implementar | Muitos usuários simultâneos podem travar o ERP |

**Quando sugerir:** poucos usuários, dados leves, cliente aceita carga no ERP, sistema fonte estável e rápido.

**Implementação técnica:**
- SF tipo SQL com `jdbcId` apontando para a conexão JDBC externa
- Ou SF tipo INTEGRATION para APIs REST
- Usar Tabelas Online (`createOnlineTableMitra`) para centralizar queries complexas

#### Modelo B — Importação Periódica
```
Cron dispara → Data Loader importa para IMP_ → SF faz upsert na tabela final → Dashboard lê tabela local
```

| Prós | Contras |
|------|---------|
| Dashboard rápido (lê banco local) | Dados com delay (depende do cron) |
| Protege o ERP de carga excessiva | Precisa de rotina e monitoramento |
| Funciona mesmo se a fonte cair | Armazenamento duplicado |
| Escala para muitos usuários | Mais complexo de implementar |

**Quando sugerir:** muitos usuários, dashboards pesados, ERP sensível a carga, tolerância a delay.

#### Modelo C — Misto
```
KPIs leves → Online (API/JDBC direto)
Grids/tabelas pesadas → Importação periódica
```

| Prós | Contras |
|------|---------|
| Equilíbrio entre atualização e performance | Mais complexo de manter |
| Dados críticos em real-time, pesados em batch | Duas estratégias para gerenciar |

**Quando sugerir:** quando parte dos dados é leve e precisa ser real-time (contadores, status), mas outra parte é pesada (grids com milhares de linhas).

### 2.2 Subtipos de Fonte — Perguntas Adicionais

#### Se a fonte é banco de dados (JDBC):
- [ ] Qual o tipo? (MySQL, PostgreSQL, Oracle, SQL Server?)
- [ ] O banco é acessível pela internet ou precisa de tunnel?
- [ ] Existe usuário read-only?
- [ ] Quais as queries/views que trazem os dados desejados?

**Fluxo obrigatório para descobrir as queries:**
1. **Perguntar primeiro:** "Você já tem as queries SQL prontas ou quer que eu ajude a construir?"
2. **Se o usuário tem queries:** usar as que ele fornecer. Validar com `LIMIT 10` antes de criar o Data Loader.
3. **Se o usuário NÃO tem queries:** perguntar: "Quer que eu pesquise na documentação oficial do [nome do ERP/sistema] as queries e tabelas adequadas para extrair esses dados?"
   - Se sim: buscar na internet a documentação do sistema (ex: "Sankhya tabelas financeiro", "TOTVS Protheus tabela SE1", "SAP BKPF BSEG")
   - Apresentar ao usuário o que encontrou e validar antes de usar
   - Se não encontrar docs: pedir para o usuário consultar o time de TI do ERP ou explorar o schema do banco juntos

> **IMPORTANTE:** Nunca inventar queries baseado em suposições. Se não tem certeza da estrutura, SEMPRE validar com o usuário ou pesquisar na documentação oficial.

#### Se a fonte é API REST:
- [ ] Existe template de integração Mitra disponível? (listar com `listIntegrationTemplatesMitra`)
- [ ] Qual a autenticação? (API Key, Bearer Token, OAuth2, Basic Auth?)
- [ ] Qual o rate limit da API? (quantas chamadas por minuto/hora?)
- [ ] Existe paginação? Qual o limite por chamada?
- [ ] **Qual endpoint será utilizado?** Alinhar com o usuário — diferentes endpoints têm diferentes limites e comportamentos (ex: na Sankhya, o endpoint `DbExplorerSP.executeQuery` tem limite de 5.000 linhas por chamada, mas outros endpoints podem não ter essa restrição)
- [ ] Qual o endpoint que retorna os dados? Existe endpoint de listagem com filtros?

#### Se a fonte é CSV:
- [ ] **Quem faz o upload?** (o próprio usuário via tela, ou processo automatizado?)
- [ ] **Qual a frequência?** (uma vez, mensal, semanal, diário?)
- [ ] **O arquivo terá sempre o mesmo nome?** (importante para LOAD DATA)
- [ ] **O CSV novo deve SUBSTITUIR os dados anteriores ou ACUMULAR?**
- [ ] **Existe alguma chave de segmentação?** (ex: cada gestor importa seu CR, cada filial importa seus dados)
- [ ] **Qual o encoding e delimitador?** (UTF-8? Latin1? Vírgula? Ponto-e-vírgula? Tab?)
- [ ] **Qual o volume por arquivo?** (linhas e tamanho em MB)

> **Exemplo concreto CSV:** Sistema de planejamento orçamentário. Cada gestor importa um CSV por mês com dados do seu Centro de Resultado (CR). O novo arquivo deve substituir apenas os dados daquele CR naquele mês, mantendo os de outros gestores.
>
> **Solução:** Tabela com colunas `MES`, `ANO`, `CR` como chave composta. O fluxo de import faz `DELETE WHERE MES = X AND ANO = Y AND CR = Z` + `INSERT` dos novos registros, **dentro de uma transação** para evitar janelas sem dados.

---

## Fase 3 — Alinhamento Técnico Detalhado

> Esta fase é a mais crítica. A IA DEVE questionar o usuário sobre CADA item abaixo e registrar as decisões.

### 3.1 Queries e Mapeamento de Dados

- [ ] **Quais queries/endpoints serão usados?** (o usuário fornece? vamos descobrir a partir da estrutura do banco?)
- [ ] **Quais colunas são necessárias?** (trazer tudo `SELECT *` ou apenas colunas específicas?)
- [ ] **Como mapear os nomes?** (CODPARC → ID_PARCEIRO? manter nomes originais?)
- [ ] **Quais os tipos corretos?** (INT, VARCHAR, DECIMAL, DATE, DATETIME?)
- [ ] **Existem JOINs necessários?** (dados vêm de várias tabelas da fonte?)

#### 3.1.1 Teste Prévio Obrigatório das Queries

> **REGRA OBRIGATÓRIA:** Antes de criar qualquer Data Loader ou SF de importação, a IA DEVE executar a query da fonte com `LIMIT 10` (ou equivalente) para validar que retorna o esperado.

**Fluxo obrigatório:**
1. Executar a query com LIMIT para inspecionar os dados reais
2. Apresentar ao usuário: "A query retorna essas colunas com esses tipos: [lista]. Está correto?"
3. Só depois de confirmado: criar o Data Loader

**Exemplo via SF SQL temporária para teste (ou via Data Loader com `runWhenCreate: false` e depois inspecionar a IMP_):**
```sql
-- Testar query da fonte via SF SQL com jdbcId externo
SELECT * FROM PEDIDOS_ERP WHERE 1=1 LIMIT 10
```

#### 3.1.2 Mapeamento e Validação de Tipos (Cenário Sample → Produção)

> **CENÁRIO CRÍTICO:** Se o projeto já tem tabelas com dados sample, é OBRIGATÓRIO validar a compatibilidade de tipos antes de importar dados reais. Ignorar isso causa bugs silenciosos (ex: IDs não populados porque a coluna era INT mas a query retorna VARCHAR).

**Fluxo obrigatório quando existem tabelas sample:**

1. **Listar estrutura das tabelas existentes:**
```sql
-- Via SF SQL no banco do projeto (jdbcId = 1)
DESCRIBE CLIENTES;
-- ou
SELECT COLUMN_NAME, DATA_TYPE, CHARACTER_MAXIMUM_LENGTH, IS_NULLABLE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'CLIENTES'
ORDER BY ORDINAL_POSITION;
```

2. **Comparar com o retorno da query da fonte (do LIMIT 10):**

```
┌─────────────────────────────────────────────────────────────────┐
│  DE-PARA: Query Fonte → Tabela Existente                       │
├─────────────┬──────────────┬──────────────┬────────────────────┤
│ Coluna ERP  │ Tipo ERP     │ Coluna Mitra │ Tipo Mitra         │
├─────────────┼──────────────┼──────────────┼────────────────────┤
│ CODPARC     │ VARCHAR(20)  │ ID           │ INT  ⚠️ INCOMP.    │
│ RAZAO       │ VARCHAR(200) │ NOME         │ VARCHAR(100) ⚠️    │
│ DT_CAD      │ DATETIME     │ DT_CADASTRO  │ DATE  ⚠️ PREC.    │
│ ATIVO       │ CHAR(1)      │ (não existe) │ — CRIAR            │
└─────────────┴──────────────┴──────────────┴────────────────────┘
```

3. **Apresentar ao usuário e pedir confirmação:**
   - "Encontrei essas incompatibilidades de tipo. Posso ajustar as tabelas existentes com ALTER TABLE?"
   - "A coluna CODPARC é VARCHAR no ERP mas INT na sua tabela. Posso alterar para VARCHAR(20)?"
   - "A coluna ATIVO não existe na sua tabela. Posso adicioná-la?"

4. **Executar os ajustes ANTES de importar:**
```javascript
// Ajustar tipos incompatíveis
await runDdlMitra({ projectId, sql: 'ALTER TABLE CLIENTES MODIFY ID VARCHAR(20);' });

// Adicionar colunas faltantes
await runDdlMitra({ projectId, sql: 'ALTER TABLE CLIENTES ADD COLUMN ATIVO CHAR(1) DEFAULT "S";' });
```

5. **Limpar dados sample (com confirmação do usuário):**
```javascript
// SOMENTE após confirmação explícita do usuário
await runDmlMitra({ projectId, sql: 'DELETE FROM CLIENTES;' });
await runDmlMitra({ projectId, sql: 'DELETE FROM PEDIDOS;' });
// etc.
```

> **PROIBIDO:**
> - Nunca ignorar incompatibilidades de tipo silenciosamente. Se a coluna da fonte é VARCHAR e a tabela é INT, a importação pode falhar ou perder dados SEM erro visível.
> - Nunca deletar dados sample sem perguntar ao usuário.
> - Nunca pular colunas da query fonte sem informar o usuário (ex: "a query retorna CODPARC mas não vou importar porque não tem coluna correspondente" — ERRADO, deve perguntar).

### 3.2 Estratégia de Registros Deletados

> **PERGUNTA OBRIGATÓRIA:** "Como vamos tratar registros que foram excluídos no sistema fonte?"

A IA DEVE apresentar as 3 opções e discutir cada uma:

| Estratégia | Como funciona | Quando usar |
|-----------|---------------|-------------|
| **Ignorar deletes** | Registros deletados na fonte permanecem na Mitra | Dados históricos que nunca devem sumir (ex: vendas passadas) |
| **Soft Delete (flag)** | Coluna `ATIVO = 0` ou `DELETADO_EM = timestamp` | Quando precisa saber o que foi deletado e quando |
| **Full Refresh periódico** | Trunca e reimporta tudo periodicamente | Tabelas pequenas/médias, garantia de consistência total |
| **Comparação incremental** | Compara IDs da fonte vs local, deleta os ausentes | Tabelas grandes onde full refresh é inviável |

**Perguntas de follow-up:**
- [ ] O sistema fonte tem campo de "data de exclusão" ou flag de inativo?
- [ ] Os registros são realmente deletados (hard delete) ou apenas inativados no ERP?
- [ ] Qual o impacto de manter registros fantasmas? (dashboards mostrando dados que não existem mais?)
- [ ] Se full refresh: qual a janela de tempo aceitável para rodar? (volume determina duração)

### 3.3 Estratégia de Atualização Incremental

> **PERGUNTA OBRIGATÓRIA:** "Existe alguma coluna que indica quando um registro foi criado ou alterado pela última vez?"

- [ ] **Coluna de cursor:** `DT_ALTERACAO`, `UPDATED_AT`, `DT_INCLUSAO`, `MODIFIED_DATE`?
- [ ] **Tipo da coluna:** DATETIME? TIMESTAMP? BIGINT (epoch)?
- [ ] **Ela é confiável?** (sempre atualizada quando o registro muda? Ou só na criação?)
- [ ] **Ela tem índice no banco fonte?** (crucial para performance da query incremental)

**Se não existe coluna de cursor:**
- Sugerir full refresh em intervalos adequados ao volume
- OU sugerir que o cliente peça ao time de TI do ERP para criar essa coluna/trigger
- Documentar o impacto: sem cursor = full refresh toda vez = mais lento e mais carga

**Template de query incremental:**
```sql
-- Data Loader query com cursor
SELECT * FROM TABELA_FONTE
WHERE DT_ALTERACAO >= '{{ultima_sync}}'
ORDER BY DT_ALTERACAO
```

### 3.4 Frequência e Agendamento (Cron)

A IA DEVE sugerir um padrão e perguntar se atende:

**Sugestão padrão para importação periódica:**
- **Incremental:** a cada 30 minutos durante horário comercial → `0 */30 6-22 * * *`
- **Full refresh:** 1x por dia de madrugada → `0 0 3 * * *` (variar hora para não sobrecarregar)
- **Apenas horário comercial?** Ou 24/7?

Perguntas:
- [ ] O cron pode rodar fora do horário comercial? (madrugada para full syncs)
- [ ] Existe algum horário que o ERP fica indisponível? (backup, manutenção?)
- [ ] Qual o delay máximo aceitável para os dados? (isso define o intervalo do cron)

> **Lembrete Spring Cron:** Mitra usa 6 campos: `segundo minuto hora diaMes mes diaSemana`. Sempre iniciar com `0`.

### 3.5 Volumetria e Batching

> **REGRA OBRIGATÓRIA:** Para volumes grandes, SEMPRE usar Data Loader com parâmetros para importar em lotes. NUNCA executar uma query gigantesca de uma vez no banco do cliente.

- [ ] **Qual o volume total da primeira carga?** (ex: 5 milhões de registros de NFs dos últimos 5 anos)
- [ ] **Qual a estratégia de lote?** (por mês? por filial? por faixa de ID?)

**Exemplo de batching por mês:**
```javascript
// Data Loader query parametrizada
"SELECT * FROM NFS WHERE MONTH(DT_EMISSAO) = {{mes}} AND YEAR(DT_EMISSAO) = {{ano}}"

// SF JavaScript que orquestra os lotes
for (let ano = 2020; ano <= 2025; ano++) {
  for (let mes = 1; mes <= 12; mes++) {
    await executeDataLoaderMitra({ projectId, dataLoaderId, input: { mes, ano } });
    // ... upsert da IMP_ para tabela final ...
  }
}
```

- [ ] **Existe limite de linhas por query no sistema fonte?** (Sankhya = 5.000 por chamada)
- [ ] **O banco do cliente aguenta queries que retornam muitos registros de uma vez?**

### 3.6 Monitoramento e Logs

> **REGRA OBRIGATÓRIA:** Toda integração DEVE ter uma tela de monitoramento de logs.

Perguntas:
- [ ] Quem precisa ver os logs? (só TI ou gestores também?)
- [ ] Quer notificação em caso de falha? (via variável + SF que verifica status?)
- [ ] Quer ver estatísticas? (tempo de execução, quantidade de registros importados, última atualização?)

### 3.7 Atomicidade e Consistência

> **ALERTA CRÍTICO:** Nunca fazer DELETE + INSERT separados sem transação! Se um usuário acessar a tela entre o DELETE e o INSERT, verá dados vazios.

- [ ] O fluxo de atualização vai deletar e reinserir? Se sim, garantir atomicidade (ver seção Boas Práticas)
- [ ] Existe risco de concorrência? (dois crons rodando ao mesmo tempo?)

---

## Fase 4 — Checkpoint de Consenso

> **REGRA ABSOLUTA:** A IA DEVE gerar este resumo e pedir confirmação EXPLÍCITA antes de codar.

### Template de Contrato de Integração

A IA deve preencher e apresentar ao usuário:

```
╔══════════════════════════════════════════════════════════╗
║            CONTRATO DE INTEGRAÇÃO — RESUMO              ║
╠══════════════════════════════════════════════════════════╣
║                                                          ║
║  FONTE: [nome do sistema/ERP]                           ║
║  TIPO DE ACESSO: [JDBC / API / CSV]                     ║
║  MODELO: [Online / Import Periódico / Misto]            ║
║                                                          ║
║  TABELAS/ENTIDADES:                                     ║
║  ┌──────────────┬──────────┬────────────┬─────────────┐ ║
║  │ Entidade     │ Volume   │ Cursor     │ Deletes     │ ║
║  ├──────────────┼──────────┼────────────┼─────────────┤ ║
║  │ Clientes     │ ~10k     │ DT_ALT     │ Soft delete │ ║
║  │ Pedidos      │ ~500k    │ DT_MOD     │ Full daily  │ ║
║  │ Produtos     │ ~2k      │ nenhum     │ Full 1x/dia │ ║
║  └──────────────┴──────────┴────────────┴─────────────┘ ║
║                                                          ║
║  FREQUÊNCIA:                                            ║
║  - Incremental: a cada 30 min (6h-22h)                  ║
║  - Full refresh: diário às 3h                           ║
║                                                          ║
║  BATCHING: por mês/ano para carga inicial               ║
║                                                          ║
║  DELETADOS: Soft delete com coluna ATIVO                ║
║                                                          ║
║  MONITORAMENTO: Tela de logs com status por execução    ║
║                                                          ║
║  DASHBOARD: "Última atualização: há X min" visível      ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝

Confirma que posso implementar seguindo EXATAMENTE este plano?
```

> **PROIBIDO:** Começar a implementar sem a confirmação explícita do usuário neste checkpoint. Se o usuário tiver qualquer dúvida, VOLTA para a fase relevante.

---

## Fase 5 — Implementação

> Somente após confirmação no Checkpoint da Fase 4.

### 5.1 Fluxo de Implementação — Import Periódico (JDBC)

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│  1. Criar JDBC  │────▶│  2. Criar Data   │────▶│  3. Criar tabelas   │
│  Connection     │     │  Loader(s)       │     │  finais (DDL)       │
└─────────────────┘     └──────────────────┘     └─────────────────────┘
                                                          │
┌─────────────────┐     ┌──────────────────┐              │
│  6. Criar tela  │◀────│  5. Configurar   │◀─────────────┘
│  de logs        │     │  cron            │     ┌─────────────────────┐
└─────────────────┘     └──────────────────┘◀────│  4. Criar SF        │
                                                  │  JavaScript de      │
                                                  │  orquestração       │
                                                  └─────────────────────┘
```

**Passo 1 — Conexão JDBC:**
```javascript
await createJdbcConnectionMitra({
  projectId,
  name: 'ERP Cliente',
  type: 'mysql',           // mysql | postgres | oracle | sqlserver
  host: 'erp.cliente.com',
  port: 3306,
  database: 'erp_prod',
  user: 'readonly',
  password: 'senha'
});
```
Se on-premise: usar Cloudflare Tunnel antes (`createTunnelMitra` → `addTunnelRouteMitra` → usuário configura JDBC pela UI).

**Passo 2 — Data Loader(s):**
```javascript
// SEMPRE com parâmetros para batching quando volume for grande
await createDataLoaderMitra({
  projectId,
  jdbcId: 2,
  name: 'Importar Pedidos',
  query: `SELECT ID, CLIENTE_ID, VALOR, STATUS, DT_EMISSAO, DT_ALTERACAO
          FROM PEDIDOS
          WHERE DT_ALTERACAO >= '{{ultima_sync}}'
          AND MONTH(DT_EMISSAO) = {{mes}}
          AND YEAR(DT_EMISSAO) = {{ano}}`,
  runWhenCreate: false
});
```

**Passo 3 — DDL das tabelas finais:**
```javascript
await runDdlMitra({
  projectId,
  sql: `CREATE TABLE IF NOT EXISTS PEDIDOS (
    ID INT PRIMARY KEY,
    CLIENTE_ID INT,
    VALOR DECIMAL(15,2),
    STATUS VARCHAR(50),
    DT_EMISSAO DATETIME,
    DT_ALTERACAO DATETIME,
    IMPORTADO_EM TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_dt_alteracao (DT_ALTERACAO),
    INDEX idx_cliente (CLIENTE_ID)
  );`
});
```

**Passo 4 — SF JavaScript de orquestração:**
```javascript
const code = `
const sdk = require('mitra-sdk');
const projectId = ${projectId};

// 1. Buscar última sync
const lastSync = await sdk.getVariableMitra({ projectId, key: 'PEDIDOS_LAST_SYNC' });
const ultimaSync = lastSync?.result?.value || '2000-01-01 00:00:00';
const agora = new Date().toISOString().slice(0, 19).replace('T', ' ');

// 2. Executar Data Loader incremental
await sdk.executeDataLoaderMitra({
  projectId,
  dataLoaderId: ${dataLoaderId},
  input: { ultima_sync: ultimaSync, mes: new Date().getMonth() + 1, ano: new Date().getFullYear() }
});

// 3. Upsert atômico: INSERT ... ON DUPLICATE KEY UPDATE (sem DELETE!)
await sdk.runDmlMitra({
  projectId,
  sql: \`
    INSERT INTO PEDIDOS (ID, CLIENTE_ID, VALOR, STATUS, DT_EMISSAO, DT_ALTERACAO)
    SELECT ID, CLIENTE_ID, VALOR, STATUS, DT_EMISSAO, DT_ALTERACAO
    FROM IMP_IMPORTAR_PEDIDOS
    ON DUPLICATE KEY UPDATE
      CLIENTE_ID = VALUES(CLIENTE_ID),
      VALOR = VALUES(VALOR),
      STATUS = VALUES(STATUS),
      DT_EMISSAO = VALUES(DT_EMISSAO),
      DT_ALTERACAO = VALUES(DT_ALTERACAO),
      IMPORTADO_EM = CURRENT_TIMESTAMP
  \`
});

// 4. Atualizar variável de última sync
await sdk.setVariableMitra({ projectId, key: 'PEDIDOS_LAST_SYNC', value: agora });

// 5. Log personalizado
await sdk.runDmlMitra({
  projectId,
  sql: \`INSERT INTO LOG_IMPORTACOES (ENTIDADE, TIPO, STATUS, REGISTROS, EXECUTADO_EM)
         VALUES ('PEDIDOS', 'INCREMENTAL', 'SUCESSO',
                 (SELECT COUNT(*) FROM IMP_IMPORTAR_PEDIDOS),
                 NOW())\`
});

return { status: 'ok', registros_processados: 'ver log' };
`;

await createServerFunctionMitra({
  projectId,
  name: 'cron_importar_pedidos',
  type: 'JAVASCRIPT',
  code,
  description: 'Importação incremental de pedidos do ERP'
});
```

**Passo 5 — Configurar Cron:**
```javascript
await updateServerFunctionMitra({
  projectId,
  serverFunctionId: sfId,
  cronExpression: '0 */30 6-22 * * *'  // a cada 30min das 6h às 22h
});
```

**Passo 6 — Tela de Logs:** Ver seção [Boas Práticas — Tela de Logs](#63-tela-de-monitoramento-de-logs-obrigatória).

---

### 5.2 Fluxo de Implementação — CSV Recorrente

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│  1. Criar DDL   │────▶│  2. Criar SF SQL │────▶│  3. Criar tela      │
│  tabela final   │     │  com LOAD DATA   │     │  de upload          │
└─────────────────┘     └──────────────────┘     └─────────────────────┘
```

**Fluxo no frontend (React):**
```typescript
import {
  uploadFileLoadableMitra,
  executeServerFunctionMitra
} from 'mitra-interactions-sdk';

// 1. Upload do CSV
await uploadFileLoadableMitra({ projectId, file: arquivoCsv });

// 2. Executar SF que faz LOAD DATA + lógica de negócio
await executeServerFunctionMitra({
  projectId,
  serverFunctionId: sfLoadCsvId,
  input: { mes: 3, ano: 2025, cr: 'CR001' }
});
```

**SF SQL para LOAD DATA com segmentação:**
```sql
-- Primeiro: deletar dados antigos DAQUELE segmento (atômico na mesma SF)
DELETE FROM ORCAMENTO WHERE MES = {{mes}} AND ANO = {{ano}} AND CR = '{{cr}}';

-- Segundo: carregar novos dados do CSV
LOAD DATA LOCAL INFILE '${LOADABLE_FILE:arquivo.csv}'
INTO TABLE ORCAMENTO
CHARACTER SET utf8
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(CONTA, DESCRICAO, VALOR_PLANEJADO, VALOR_REALIZADO);

-- Terceiro: atualizar colunas de contexto nos registros recém-importados (que não têm MES/ANO/CR)
UPDATE ORCAMENTO SET MES = {{mes}}, ANO = {{ano}}, CR = '{{cr}}'
WHERE MES IS NULL;
```

> **Nota sobre atomicidade CSV:** Como a SF tipo SQL executa todos os statements em sequência na mesma conexão, o DELETE + LOAD DATA + UPDATE ocorrem de forma atômica o suficiente para CSVs. Para volumes muito grandes, considerar tabela staging + upsert.

---

### 5.3 Fluxo de Implementação — API REST (Integration)

```
┌──────────────┐     ┌───────────────┐     ┌────────────────┐     ┌────────────┐
│ 1. Listar    │────▶│ 2. Testar     │────▶│ 3. Criar       │────▶│ 4. Criar   │
│ templates    │     │ conexão       │     │ Integration    │     │ SF tipo    │
│              │     │               │     │                │     │ INTEGRATION│
└──────────────┘     └───────────────┘     └────────────────┘     └────────────┘
```

```javascript
// 1. Descobrir templates disponíveis
const templates = await listIntegrationTemplatesMitra();

// 2. Testar
const teste = await testIntegrationMitra({
  blueprintId: 'api_key',
  credentials: { base_url: 'https://api.erp.com', api_key: 'xxx' },
  testEndpoint: '/health'
});

// 3. Criar integração
const integ = await createIntegrationMitra({
  projectId,
  name: 'ERP API',
  slug: 'erp-api',
  blueprintId: 'api_key',
  blueprintType: 'HTTP_REQUEST',
  authType: 'STATIC_KEY',
  credentials: { base_url: 'https://api.erp.com', api_key: 'xxx' }
});

// 4. Criar SF tipo INTEGRATION
await createServerFunctionMitra({
  projectId,
  name: 'buscar_clientes_api',
  type: 'INTEGRATION',
  code: JSON.stringify({
    connection: 'erp-api',
    method: 'POST',
    endpoint: '/api/query',
    body: {
      query: "SELECT * FROM CLIENTES WHERE DT_ALT >= 'event.ultimaSync'"
    }
  }),
  description: 'Busca clientes na API do ERP'
});
```

**Para APIs com paginação (ex: Sankhya endpoint `DbExplorerSP.executeQuery` com limite de 5.000 linhas — alinhar com o usuário qual endpoint será utilizado, pois cada um tem limites diferentes):**
```javascript
// SF JAVASCRIPT que orquestra paginação
const code = `
const sdk = require('mitra-sdk');
let offset = 0;
const limit = 5000;
let hasMore = true;

while (hasMore) {
  const result = await sdk.executeServerFunctionMitra({
    projectId: ${projectId},
    serverFunctionId: ${sfIntegrationId},
    input: { offset, limit }
  });

  const rows = result.result.output.body.rows;
  if (!rows || rows.length === 0) {
    hasMore = false;
    break;
  }

  // Inserir lote na tabela local
  // ... upsert logic ...

  offset += limit;
  if (rows.length < limit) hasMore = false;
}
`;
```

---

## Boas Práticas Obrigatórias

### 6.1 Atomicidade — Evitar Janelas Sem Dados

> **PROBLEMA:** Se o fluxo faz DELETE + INSERT e um usuário acessa a tela entre os dois comandos, ele vê dados vazios.

**Soluções obrigatórias (escolher uma):**

**Opção A — INSERT ON DUPLICATE KEY UPDATE (preferida):**
```sql
INSERT INTO TABELA_FINAL (ID, COL1, COL2)
SELECT ID, COL1, COL2 FROM IMP_STAGING
ON DUPLICATE KEY UPDATE
  COL1 = VALUES(COL1),
  COL2 = VALUES(COL2);
```
Nunca deleta, só atualiza. Zero janela sem dados.

**Opção B — Tabela shadow (para full refresh que precisa limpar deletados):**
```sql
-- 1. Popula tabela shadow
TRUNCATE TABLE PEDIDOS_SHADOW;
INSERT INTO PEDIDOS_SHADOW SELECT * FROM IMP_PEDIDOS;

-- 2. Renomeia atomicamente
RENAME TABLE PEDIDOS TO PEDIDOS_OLD, PEDIDOS_SHADOW TO PEDIDOS;

-- 3. Limpa a antiga
DROP TABLE PEDIDOS_OLD;

-- 4. Recria shadow vazia para próximo ciclo
CREATE TABLE PEDIDOS_SHADOW LIKE PEDIDOS;
```

**Opção C — Flag de versão:**
```sql
-- Importa com flag de versão nova
INSERT INTO PEDIDOS (ID, ..., VERSAO) SELECT ID, ..., 2 FROM IMP_PEDIDOS;

-- Dashboard filtra: WHERE VERSAO = (SELECT MAX(VERSAO) FROM PEDIDOS)

-- Limpa versão antiga em horário seguro
DELETE FROM PEDIDOS WHERE VERSAO < (SELECT MAX(VERSAO) FROM PEDIDOS);
```

> **PROIBIDO:** Fazer `TRUNCATE` ou `DELETE FROM tabela_final` seguido de `INSERT` em operações que rodam durante horário comercial, sem nenhuma das estratégias acima.

### 6.2 Indicador de "Última Atualização" nos Dashboards

> **REGRA OBRIGATÓRIA:** Todo dashboard que consome dados importados DEVE mostrar "Última atualização: há X minutos" visível ao usuário.

**Implementação:**

1. A SF de importação atualiza uma variável ao final:
```javascript
await sdk.setVariableMitra({
  projectId,
  key: 'ULTIMA_SYNC_PEDIDOS',
  value: new Date().toISOString()
});
```

2. O frontend lê e exibe:
```typescript
import { getVariableMitra } from 'mitra-interactions-sdk';

const v = await getVariableMitra({ projectId, key: 'ULTIMA_SYNC_PEDIDOS' });
const lastSync = new Date(v.result.value);
const diffMin = Math.round((Date.now() - lastSync.getTime()) / 60000);

// Exibir: "Última atualização: há {diffMin} minutos"
// Se diffMin > threshold esperado: exibir em vermelho como alerta
```

3. **Lógica de alerta:**
   - Até 2x o intervalo do cron: normal (verde/cinza)
   - Entre 2x e 4x: atenção (amarelo)
   - Acima de 4x: alerta — importação pode estar com problema (vermelho)

### 6.3 Tela de Monitoramento de Logs (OBRIGATÓRIA)

> **REGRA OBRIGATÓRIA:** Toda integração DEVE ter uma tela de acompanhamento de logs acessível ao usuário.

**A tela deve combinar duas fontes de log:**

#### Fonte 1 — Logs do Data Loader (tabela `INT_DATALOADER_EXECUTION`)
```sql
-- Query para exibir logs de execução dos Data Loaders
SELECT
  dle.ID,
  dl.NAME AS DATA_LOADER,
  dle.STATUS,
  dle.CREATED_AT AS INICIO,
  dle.UPDATED_AT AS FIM,
  TIMESTAMPDIFF(SECOND, dle.CREATED_AT, dle.UPDATED_AT) AS DURACAO_SEG,
  dle.EXECUTION_JSON
FROM INT_DATALOADER_EXECUTION dle
JOIN INT_DATALOADER dl ON dl.ID = dle.DATALOADER_ID
ORDER BY dle.CREATED_AT DESC
LIMIT 50
```

**Campos úteis do `EXECUTION_JSON` (JSON serializado do `DataLoaderExecutionDTO`):**
- Tempos de execução
- Metadados do CSV (tamanho, quantidade de linhas)
- Statements SQL gerados
- Status/erros detalhados
- Variáveis de contexto e info do usuário

#### Fonte 2 — Log personalizado (tabela própria)
```sql
CREATE TABLE LOG_IMPORTACOES (
  ID INT AUTO_INCREMENT PRIMARY KEY,
  ENTIDADE VARCHAR(100),        -- 'PEDIDOS', 'CLIENTES', etc.
  TIPO VARCHAR(50),             -- 'INCREMENTAL', 'FULL', 'CSV'
  STATUS VARCHAR(50),           -- 'SUCESSO', 'ERRO', 'PARCIAL'
  REGISTROS_IMPORTADOS INT,
  REGISTROS_ATUALIZADOS INT,
  REGISTROS_DELETADOS INT,
  MENSAGEM_ERRO TEXT,
  EXECUTADO_EM DATETIME DEFAULT NOW(),
  DURACAO_MS INT,
  PARAMETROS JSON               -- ex: {"mes": 3, "ano": 2025}
);
```

**A SF de importação DEVE alimentar esta tabela:**
```javascript
const inicio = Date.now();
try {
  // ... lógica de importação ...
  const duracao = Date.now() - inicio;
  await sdk.runDmlMitra({
    projectId,
    sql: `INSERT INTO LOG_IMPORTACOES
          (ENTIDADE, TIPO, STATUS, REGISTROS_IMPORTADOS, DURACAO_MS)
          VALUES ('PEDIDOS', 'INCREMENTAL', 'SUCESSO', ${count}, ${duracao})`
  });
} catch (err) {
  const duracao = Date.now() - inicio;
  await sdk.runDmlMitra({
    projectId,
    sql: `INSERT INTO LOG_IMPORTACOES
          (ENTIDADE, TIPO, STATUS, MENSAGEM_ERRO, DURACAO_MS)
          VALUES ('PEDIDOS', 'INCREMENTAL', 'ERRO', '${err.message.replace(/'/g, "''")}', ${duracao})`
  });
  throw err;
}
```

**Componentes da tela de logs:**
- Grid com filtros por entidade, tipo, status e data
- Indicador visual: 🟢 sucesso, 🔴 erro, 🟡 parcial
- Detalhes expandíveis (mensagem de erro, parâmetros, EXECUTION_JSON)
- Gráfico de tempo de execução para detectar degradação
- Botão de "Executar agora" para reimportação manual

### 6.4 Data Loader — Sempre em Lotes com Parâmetros

> Veja seção **3.5 Volumetria e Batching** para o detalhamento completo e exemplos de código. Aqui reforçamos o resumo das regras:

- **Volume > 50.000 registros:** OBRIGATÓRIO usar Data Loader com parâmetros (mês, ano, faixa de ID)
- **APIs com paginação:** respeitar o limite por chamada (offset + limit) e iterar — alinhar com o usuário qual endpoint usar, pois cada um tem limites diferentes
- **Nunca** executar `SELECT * FROM TABELA_GRANDE` sem parâmetros de lote

---

## Anti-Patterns (Proibições)

> **PROIBIDO: Importar via loop de INSERTs**
> Nunca criar SF JAVASCRIPT que faz SELECT no banco externo e depois INSERT um por um em loop. Use Data Loader (JDBC) ou LOAD DATA (CSV).

> **PROIBIDO: Query sem limites no banco do cliente**
> Nunca executar `SELECT * FROM TABELA_GRANDE` sem parâmetros de lote no Data Loader. Sempre usar parâmetros (mês, ano, faixa de ID) para limitar cada execução.

> **PROIBIDO: DELETE + INSERT sem atomicidade**
> Nunca fazer `TRUNCATE/DELETE` seguido de `INSERT` em tabelas que estão sendo consultadas por dashboards em horário comercial. Usar `ON DUPLICATE KEY UPDATE`, tabela shadow ou flag de versão.

> **PROIBIDO: Codar antes do Checkpoint**
> Nunca iniciar implementação de integração sem ter completado TODAS as Fases 1-4 e recebido confirmação explícita do usuário.

> **PROIBIDO: Dashboard de importação sem logs**
> Nunca entregar uma integração sem tela de monitoramento de logs.

> **PROIBIDO: Dashboard de importação sem "Última atualização"**
> Nunca entregar um dashboard que consome dados importados sem o indicador visível de última atualização.

> **PROIBIDO: SFs temporárias**
> Nunca criar e deletar Server Functions dinamicamente por execução.

> **PROIBIDO: fetch/axios em SF JavaScript**
> Nunca usar fetch ou axios dentro de SF JAVASCRIPT para chamar APIs externas. Usar SF tipo INTEGRATION. Para APIs Mitra, usar `require('mitra-sdk')`.

> **PROIBIDO: Importar sem testar a query antes**
> Nunca criar Data Loader e executar diretamente sem antes rodar a query com LIMIT 10 para validar colunas, tipos e dados retornados. Sempre mostrar o resultado ao usuário antes de prosseguir.

> **PROIBIDO: Ignorar incompatibilidade de tipos silenciosamente**
> Se a tabela existente tem coluna INT mas a query retorna VARCHAR (ou qualquer incompatibilidade), a IA DEVE informar o usuário e propor ALTER TABLE. Nunca prosseguir esperando que "vai funcionar".

> **PROIBIDO: Deixar dados sample ao importar dados reais**
> Quando o projeto transiciona de dados sample para integração real, a IA DEVE perguntar se pode deletar os dados fictícios. Nunca misturar dados sample com dados reais importados.

> **PROIBIDO: Inventar queries sem validação**
> Se a IA não tem certeza da estrutura do banco do ERP/sistema fonte, NUNCA inventar nomes de tabelas ou colunas. Perguntar ao usuário, pesquisar na documentação oficial, ou explorar o schema juntos.

---

## Referência SDK

### Funções de Data Loader (backend: `mitra-sdk`)

| Função | Uso |
|--------|-----|
| `createDataLoaderMitra({ projectId, jdbcId, name, query, runWhenCreate })` | Cria Data Loader. Gera tabela `IMP_<name>` |
| `executeDataLoaderMitra({ projectId, dataLoaderId, input? })` | Executa (trunca IMP_ e reimporta). Disponível também no frontend |
| `updateDataLoaderMitra({ projectId, dataLoaderId, ... })` | Atualiza query/config |
| `deleteDataLoaderMitra({ projectId, dataLoaderId })` | Remove Data Loader |

### Funções de Server Function (backend: `mitra-sdk`)

| Função | Uso |
|--------|-----|
| `createServerFunctionMitra({ projectId, name, type, code, description, jdbcId? })` | Cria SF (SQL/INTEGRATION/JAVASCRIPT) |
| `updateServerFunctionMitra({ projectId, serverFunctionId, code?, cronExpression?, ... })` | Atualiza SF. Cron: Spring 6 campos |
| `executeServerFunctionMitra({ projectId, serverFunctionId, input? })` | Executa (sync). **Usar `input`, NÃO `params`** |
| `executeServerFunctionAsyncMitra({ projectId, serverFunctionId, input? })` | Executa (async, para >30s) |
| `getServerFunctionExecutionMitra({ projectId, executionId })` | Polling de execução async |
| `togglePublicExecutionMitra({ projectId, serverFunctionId, publicExecution })` | Torna SF pública/privada |

### Funções de Integração (backend: `mitra-sdk`)

| Função | Uso |
|--------|-----|
| `listIntegrationTemplatesMitra()` | Lista templates disponíveis |
| `getIntegrationTemplateMitra({ templateId })` | Detalhes de um template |
| `testIntegrationMitra({ blueprintId?, integrationId?, credentials?, testEndpoint? })` | Testa conexão |
| `createIntegrationMitra({ projectId, name, slug, blueprintId, blueprintType, authType, credentials, ... })` | Cria integração |
| `callIntegrationMitra({ projectId, integrationSlug, method, endpoint, params?, body? })` | Chama API via integração criada |

### Funções de JDBC (backend: `mitra-sdk`)

| Função | Uso |
|--------|-----|
| `createJdbcConnectionMitra({ projectId, name, type, host, port, database, user, password })` | Cria conexão JDBC |
| `updateJdbcConnectionMitra({ projectId, jdbcId, ... })` | Atualiza conexão |

### Funções de DDL/DML (backend: `mitra-sdk`)

| Função | Uso |
|--------|-----|
| `runDdlMitra({ projectId, sql })` | CREATE TABLE, ALTER, DROP. **Param é `sql:`, nunca `ddl:`** |
| `runDmlMitra({ projectId, sql })` | INSERT, UPDATE, DELETE, LOAD DATA. **Param é `sql:` (ou `code:` para LOAD DATA)** |

### Funções de Query (frontend: `mitra-interactions-sdk`)

| Função | Uso |
|--------|-----|
| `runQueryMitra({ projectId, sql })` | SELECT only. Retorna `{ rows, rowCount }`. Colunas em UPPERCASE |

### Funções de Variáveis (ambos: `mitra-sdk` e `mitra-interactions-sdk`)

| Função | Uso |
|--------|-----|
| `setVariableMitra({ projectId, key, value })` | Define variável persistente |
| `getVariableMitra({ projectId, key })` | Lê variável |
| `listVariablesMitra({ projectId })` | Lista todas |
| `deleteVariableMitra({ projectId, key })` | Remove variável |

### Funções de Upload (frontend: `mitra-interactions-sdk`)

| Função | Uso |
|--------|-----|
| `uploadFileLoadableMitra({ projectId, file })` | Upload para pasta LOADABLE (para LOAD DATA) |
| `uploadFilePublicMitra({ projectId, file })` | Upload público (imagens, logos) |

### Tabelas Online (backend: `mitra-sdk`)

| Função | Uso |
|--------|-----|
| `createOnlineTableMitra({ projectId, jdbcId, name, sqlQuery })` | Cria Tabela Online parametrizada |
| `updateOnlineTableMitra({ projectId, onlineTableId, ... })` | Atualiza |
| `listOnlineTablesMitra({ projectId })` | Lista existentes (chamar ANTES de criar SFs) |

### Cron — Spring Cron 6 campos

```
segundo minuto hora diaMes mes diaSemana
  0      */30   6-22   *    *      *       → a cada 30min das 6h às 22h
  0       0      3     *    *      *       → diário às 3h
  0       0     */2    *    *      *       → a cada 2h
  0      */5     *     *    *      *       → a cada 5min (mínimo permitido)
```

---

## Diagrama Resumo do Fluxo Completo

```
╔═══════════════════════════════════════════════════════════════╗
║                                                               ║
║   FASE 1: DISCOVERY                                          ║
║   ├── Fonte? Volume? Frequência? Tolerância?                 ║
║   └── TODAS as perguntas respondidas? ──NO──▶ VOLTA          ║
║                    │ YES                                      ║
║                    ▼                                          ║
║   FASE 2: DECISÃO DE ARQUITETURA                             ║
║   ├── Apresentar 3 modelos com prós/contras                  ║
║   ├── Subperguntas por tipo de fonte                         ║
║   └── Usuário escolheu modelo? ──NO──▶ DISCUTIR MAIS        ║
║                    │ YES                                      ║
║                    ▼                                          ║
║   FASE 3: ALINHAMENTO TÉCNICO                                ║
║   ├── 3.1 Queries e mapeamento                               ║
║   ├── 3.2 Deletados (OBRIGATÓRIO discutir)                   ║
║   ├── 3.3 Incrementalidade (OBRIGATÓRIO: coluna cursor)      ║
║   ├── 3.4 Cron/frequência                                    ║
║   ├── 3.5 Batching/volume                                    ║
║   ├── 3.6 Logs e monitoramento                               ║
║   └── 3.7 Atomicidade                                        ║
║   └── TODOS alinhados? ──NO──▶ VOLTA ao item pendente        ║
║                    │ YES                                      ║
║                    ▼                                          ║
║   FASE 4: CHECKPOINT                                         ║
║   ├── Gerar Contrato de Integração                           ║
║   └── Confirmação EXPLÍCITA? ──NO──▶ VOLTA à fase relevante  ║
║                    │ YES                                      ║
║                    ▼                                          ║
║   FASE 5: IMPLEMENTAÇÃO                                      ║
║   ├── JDBC/Integration/CSV (conforme decisão)                ║
║   ├── Data Loader + SF orquestradora                         ║
║   ├── Tela de logs (OBRIGATÓRIO)                             ║
║   ├── "Última atualização" no dashboard (OBRIGATÓRIO)        ║
║   └── Otimizar: Tabelas Online + mínimo de SFs              ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```
