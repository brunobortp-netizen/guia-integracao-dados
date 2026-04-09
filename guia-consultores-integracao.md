# Guia do Consultor — Integração de Dados na Plataforma Mitra

> Este guia ensina as features de integração disponíveis na plataforma e como pedir para a IA utilizá-las. Não é preciso saber programar — basta entender os conceitos para orientar a IA corretamente.

---

## 1. Como a IA conecta seu projeto a dados externos

Existem **3 formas** de trazer dados de fora para o Mitra:

| Forma | O que faz | Exemplo de uso |
|-------|-----------|----------------|
| **Conexão JDBC** | Conecta direto no banco de dados do cliente | "Conecta no Oracle do ERP e puxa os pedidos" |
| **Integração (API)** | Conecta via API REST do sistema externo | "Integra com a API do Sankhya pra buscar clientes" |
| **Upload CSV** | Usuário sobe um arquivo pela tela | "Gestores vão importar orçamento todo mês via planilha" |

---

## 2. Os 3 modelos de consumo de dados

Antes de escolher qual ferramenta usar, é preciso decidir **como** os dados serão consumidos. Sempre discuta com o cliente:

### Modelo A — Online (tempo real)

```
Usuário abre dashboard → SF busca dados no ERP naquele momento → Dados frescos
```

| Prós | Contras |
|------|---------|
| Dados sempre atualizados | Cada acesso gera carga no ERP |
| Mais simples de implementar | Se o ERP cair, o dashboard para |

**Quando usar:** poucos usuários, dados leves, ERP robusto.

### Modelo B — Importação periódica

```
Cron roda a cada 30min → Importa dados para o Mitra → Dashboard lê dados locais (rápido)
```

| Prós | Contras |
|------|---------|
| Dashboard super rápido | Dados com delay (30min, 1h, etc.) |
| Protege o ERP | Mais complexo de montar |

**Quando usar:** muitos usuários, dashboards pesados, ERP sensível.

### Modelo C — Misto

Combina os dois: dados leves em tempo real, dados pesados importados.

> **Dica:** Um mesmo dashboard pode ter componentes usando modelos diferentes. KPIs simples em tempo real + tabela com milhares de linhas importada.

---

## 3. Os 3 tipos de Server Function (SF)

Server Functions são os "motores" que buscam e processam dados. Existem 3 tipos e a regra é simples: **usar sempre o mais simples que resolve o problema.**

### SF tipo SQL (a mais rápida — ~8ms)

Roda uma query SQL direto no banco. Use para quase tudo. Aceita **SELECT, INSERT, UPDATE, DELETE** e também comandos como **LOAD DATA** (para importar CSV). Timeout de **30 segundos**.

**Quando usar:** consultas, filtros, inserções, atualizações, importação de CSV.

**Exemplo prático:** Buscar vendas filtradas por período
```sql
SELECT VENDEDOR, SUM(VALOR) AS TOTAL
FROM VENDAS
WHERE DATA BETWEEN '{{dataInicio}}' AND '{{dataFim}}'
GROUP BY VENDEDOR
```
> Os `{{parametros}}` são preenchidos pelo frontend quando o usuário interage com filtros.

---

### SF tipo INTEGRATION (para APIs — ~500ms)

Chama uma API externa usando uma Integração já configurada.

**Quando usar:** buscar dados de ERPs, CRMs ou qualquer sistema que tenha API REST.

**Exemplo prático:** Buscar parceiros ativos no Sankhya
```
Conexão: "sankhya" (slug da integração)
Método: POST
Endpoint: /mge/service.sbr?serviceName=DbExplorerSP.executeQuery&outputType=json
Body: { sql: "SELECT CODPARC, NOME FROM TGFPAR WHERE ATIVO = 'S'" }
```
> A IA monta isso como JSON — você só precisa informar qual dado quer buscar.

---

### SF tipo JAVASCRIPT (para lógica complexa — ~2s)

Roda código JavaScript num ambiente isolado. Só usar quando os outros dois não resolvem. Timeout de **300 segundos** — por isso importações grandes precisam ser divididas em várias SFs encadeadas.

**Quando usar:** orquestrar importações em lotes, fazer loops, combinar múltiplas chamadas.

**Poder especial:** Uma SF JavaScript consegue **chamar outras SFs** (inclusive ela mesma!) para criar loops e orquestrações:

```javascript
const sdk = require('mitra-sdk');

// Chamar outra SF (ex: disparar o upsert após importar)
await sdk.executeServerFunctionMitra({
  projectId: 12345,
  serverFunctionId: 50,           // ID da SF que quero chamar
  input: { entidade: 'PEDIDOS' }  // parâmetros
});

// Chamar ela mesma para o próximo lote (cria um loop!)
await sdk.executeServerFunctionAsyncMitra({
  projectId: 12345,
  serverFunctionId: 40,           // ID desta própria SF
  input: { loteAtual: 2 }        // próximo lote
});
// Async = não espera terminar, dispara e segue (cada execução tem seus 300s)
```

**Exemplo visual de loop:**
```
SF_PROCESSAR (lote 1) → faz o trabalho → dispara ela mesma com lote 2
  SF_PROCESSAR (lote 2) → faz o trabalho → dispara ela mesma com lote 3
    SF_PROCESSAR (lote 3) → faz o trabalho → acabou, para! ✅
```

> Isso é fundamental para importações grandes — cada execução tem seus 300s de timeout, então o loop nunca estoura.

> **Regra de ouro:** Sempre prefira SQL > INTEGRATION > JAVASCRIPT. A IA já sabe disso.

---

## 4. Integrações (conexões com APIs)

Antes de criar SFs tipo INTEGRATION, é preciso configurar a **conexão** com o sistema externo.

### Como funciona

```
1. Escolher template → 2. Testar credenciais → 3. Criar integração → 4. Usar em SFs
```

**O que pedir pra IA:** "Cria uma integração com o [nome do sistema]"

A IA vai:
1. Verificar se existe um template pronto (Sankhya, TOTVS, etc.)
2. Pedir suas credenciais (API key, usuário/senha)
3. Testar a conexão
4. Criar e começar a usar

### Tipos de autenticação

| Tipo | Quando | Exemplo |
|------|--------|---------|
| **Chave fixa** (API Key, Bearer Token) | A maioria dos casos | "API key: abc123" |
| **Login dinâmico** (OAuth, sessão) | Sistemas que geram token temporário | "Sankhya OAuth com client_id e secret" |

### Sankhya — tem 3 templates

| Template | Quando usar |
|----------|-------------|
| `sankhya_gateway_mitra` | **Tentar primeiro** — apesar do nome, NÃO é o Gateway. Usa login direto com URL + usuário + senha. Funciona na maioria dos clientes |
| `sankhya_oauth` | Este sim é o **Gateway do Sankhya**. Usar quando o primeiro falha por autenticação. Precisa de `client_id`, `client_secret` e `x_token` (gerados no Portal do Desenvolvedor Sankhya) |
| `sankhya_oauth_sandbox` | Mesmas credenciais do `sankhya_oauth`, mas para ambiente de homologação/testes |

> **Dica:** Para dashboards e análise de dados no Sankhya, o melhor endpoint é o `DbExplorerSP.executeQuery`. Ele aceita SQL direto, mas retorna no máximo 5.000 linhas por chamada.

---

## 5. Tabelas Online — centralizando queries

### O que são

Uma Tabela Online é uma **query base reutilizável** que centraliza JOINs complexos. Ao invés de cada SF repetir os mesmos JOINs, todas referenciam a mesma Tabela Online.

### Exemplo prático

**Sem Tabela Online (ruim):** 3 SFs com o mesmo JOIN repetido
```sql
-- SF do card de total
SELECT SUM(VALOR) FROM VENDAS V JOIN CLIENTES C ON C.ID = V.CLIENTE_ID WHERE ...
-- SF do gráfico
SELECT MES, SUM(VALOR) FROM VENDAS V JOIN CLIENTES C ON C.ID = V.CLIENTE_ID WHERE ...
-- SF da grid
SELECT * FROM VENDAS V JOIN CLIENTES C ON C.ID = V.CLIENTE_ID WHERE ...
```

**Com Tabela Online (bom):** 1 query base, 3 SFs simples
```sql
-- Tabela Online: VW_VENDAS (criada uma vez)
SELECT V.*, C.NOME AS CLIENTE FROM VENDAS V JOIN CLIENTES C ON C.ID = V.CLIENTE_ID WHERE {{FILTROS}}

-- SF do card: só a agregação
SELECT SUM(VALOR) AS TOTAL FROM @VW_VENDAS(FILTROS="DATA >= '{{dataInicio}}'")

-- SF do gráfico: só o agrupamento
SELECT MONTH(DATA) AS MES, SUM(VALOR) FROM @VW_VENDAS(FILTROS="ANO = {{ano}}") GROUP BY MES

-- SF da grid: só os dados
SELECT * FROM @VW_VENDAS(FILTROS="1=1") LIMIT 100
```

### Como funcionam os filtros

Os `{{VARIAVEIS}}` são placeholders que cada SF preenche na hora de usar. Você pode criar **quantas variáveis quiser** na mesma Tabela Online:

```sql
-- Tabela Online com múltiplos filtros
SELECT V.*, C.NOME AS CLIENTE, F.NOME AS FILIAL
FROM VENDAS V
JOIN CLIENTES C ON C.ID = V.CLIENTE_ID
JOIN FILIAIS F ON F.ID = V.FILIAL_ID
WHERE {{FILTRO_DATA}} AND {{FILTRO_FILIAL}} AND {{FILTRO_STATUS}}
```

Cada SF preenche os filtros que precisa:
```sql
-- SF que filtra por data e filial
SELECT * FROM @VW_VENDAS(FILTRO_DATA="DATA >= '2025-01-01'", FILTRO_FILIAL="FILIAL_ID = 3", FILTRO_STATUS="1=1")

-- SF que filtra só por status (passa 1=1 nos outros)
SELECT COUNT(*) FROM @VW_VENDAS(FILTRO_DATA="1=1", FILTRO_FILIAL="1=1", FILTRO_STATUS="STATUS = 'ABERTO'")
```

> `1=1` significa "sem filtro" — é um truque SQL que sempre é verdadeiro.

### Regras importantes

- **1 Tabela Online por entidade** (ex: VW_VENDAS, VW_FINANCEIRO) — não criar uma pra cada gráfico
- **Quantos filtros quiser:** pode ter `{{FILTRO_A}}`, `{{FILTRO_B}}`, `{{FILTRO_C}}`... sem limite
- **Nunca colocar GROUP BY** dentro da Tabela Online — quem agrupa é a SF
- **Para não filtrar:** passar `1=1` como valor do filtro
- A SF que usa `@TABELA_ONLINE(...)` precisa apontar pro mesmo JDBC da Tabela Online

---

## 6. JDBC e Cloudflare Tunnel

### Conexão JDBC

O Mitra **não tem nenhuma restrição** para conectar via JDBC em qualquer banco de dados (MySQL, PostgreSQL, Oracle, SQL Server, etc.). Basta informar host, porta, banco, usuário e senha.

O que acontece na prática é que **a maioria dos bancos das empresas tem restrições de segurança** — firewalls, VPNs, redes internas fechadas. O banco do ERP do cliente normalmente não está aberto pra qualquer um acessar pela internet.

### Como resolver o acesso

Existem duas opções:

**Opção 1 — Liberar o IP do Mitra no firewall (mais simples, menos seguro)**

O cliente configura o firewall para permitir conexões vindas do IP do Mitra. Funciona, mas tem riscos: o banco fica parcialmente exposto na internet e depende do firewall estar bem configurado.

**Opção 2 — Cloudflare Tunnel (recomendado, mais seguro)**

O `cloudflared` é um pequeno programa que roda **dentro da rede do cliente**, ao lado do banco de dados. Ele abre um túnel criptografado de dentro pra fora — o Mitra se conecta pelo túnel sem que o banco precise estar exposto na internet.

```
Rede do cliente (fechada)                    Nuvem
┌──────────────────────┐                ┌──────────┐
│  Banco de dados      │                │          │
│  (Oracle, MySQL...)  │                │  Mitra   │
│         │            │                │          │
│    cloudflared ──────┼── túnel ──────▶│          │
│  (programa pequeno)  │  criptografado │          │
└──────────────────────┘                └──────────┘
   Nenhuma porta aberta!          Acesso seguro via túnel
```

**Como funciona na prática:**
1. A IA cria o túnel e gera um **token**
2. O cliente instala o `cloudflared` no servidor onde está o banco (é só um executável)
3. O `cloudflared` usa o token pra se conectar ao Mitra — o túnel fica ativo
4. O cliente configura a conexão JDBC pela interface do Mitra (com a toggle "Cloudflare" ligada)

> **Segurança:** As credenciais do banco (usuário, senha) são inseridas apenas pela interface do Mitra — nunca no chat. E o banco não precisa de nenhuma porta aberta de entrada.

---

## 7. Data Loader — importando dados do banco externo

### O que é

O Data Loader executa uma query no banco do cliente e salva o resultado numa **tabela temporária** chamada `IMP_<nome>`. Toda vez que roda, a tabela é limpa e reimportada.

### Fluxo completo

```
Data Loader roda query no ERP → preenche tabela IMP_PEDIDOS
  → SF faz upsert (INSERT/UPDATE) da IMP_PEDIDOS para tabela final PEDIDOS
    → Dashboard lê da tabela PEDIDOS (rápido, local)
```

### Quando usar

Quando muitos usuários acessam os dashboards e você **não quer que cada acesso faça uma query no ERP do cliente**. Os dados são importados periodicamente (a cada 30min, 1h, 1x por dia) e os dashboards leem do banco local do Mitra.

### Boas práticas de importação em lotes

> **Regra:** Nunca importar tudo de uma vez. Sempre usar parâmetros para dividir em lotes.

**Por quê?** Uma query `SELECT * FROM PEDIDOS` com 5 milhões de linhas pode travar o ERP do cliente.

**Como fazer:** O Data Loader aceita parâmetros como `{{mes}}` e `{{ano}}`:
```sql
-- Query do Data Loader (parametrizada)
SELECT * FROM PEDIDOS WHERE MONTH(DT_EMISSAO) = {{mes}} AND YEAR(DT_EMISSAO) = {{ano}}
```

**Como a SF JavaScript executa o Data Loader:**
```javascript
const sdk = require('mitra-sdk');

// Executa o Data Loader passando os parâmetros do lote
await sdk.executeDataLoaderMitra({
  projectId: 12345,
  dataLoaderId: 10,           // ID do Data Loader criado
  input: { mes: 3, ano: 2025 } // preenche os {{mes}} e {{ano}} da query
});

// Nesse momento a tabela IMP_IMPORTAR_PEDIDOS está preenchida
// Outra SF (separada) faz o upsert da IMP_ para a tabela final
```

A IA cria SFs que executam o Data Loader passando um mês por vez e vão avançando automaticamente:

```
Exemplo: importar pedidos de Jan/2024 a Mar/2024

SF_DL (lote 1: jan/2024)
  │  → executeDataLoaderMitra({ input: { mes: 1, ano: 2024 } })
  │  → IMP_PEDIDOS preenchida com dados de janeiro
  │
  └──▶ dispara SF_UPSERT async
        │  → INSERT INTO PEDIDOS SELECT ... FROM IMP_PEDIDOS ON DUPLICATE KEY UPDATE ...
        │  → Janeiro salvo na tabela final ✅
        │
        └──▶ dispara SF_DL async (lote 2: fev/2024)
              │  → executeDataLoaderMitra({ input: { mes: 2, ano: 2024 } })
              │  → IMP_PEDIDOS preenchida com dados de fevereiro
              │
              └──▶ dispara SF_UPSERT async
                    │  → Fevereiro salvo ✅
                    │
                    └──▶ dispara SF_DL async (lote 3: mar/2024)
                          │  → Março salvo ✅
                          │
                          └──▶ Acabou! Log de conclusão 🎉
```

Cada SF tem seus próprios **300 segundos**. Por isso são separadas — se fosse tudo junto num loop, estouraria o tempo.

### E os registros deletados no ERP?

Quando você importa dados periodicamente, um problema comum é: **o que acontece quando o cliente deleta um registro no ERP?** O registro some lá, mas continua aparecendo no Mitra.

Existem algumas formas de lidar com isso — sempre alinhe com o cliente qual preferem:

| Estratégia | Como funciona | Quando usar |
|-----------|---------------|-------------|
| **Ignorar** | Registros deletados no ERP permanecem no Mitra | Dados históricos que nunca devem sumir (ex: vendas passadas) |
| **Soft delete** | Marcar como inativo (coluna `ATIVO = 0`) ao invés de deletar | Quando precisa saber o que foi deletado |
| **Full refresh periódico** | Limpar tudo e reimportar do zero periodicamente | Tabelas pequenas/médias — garante 100% de consistência |
| **Comparar IDs** | Comparar IDs do ERP vs local, desativar os ausentes | Tabelas grandes onde reimportar tudo é inviável |

> **Dica prática:** A abordagem mais comum é combinar **incremental a cada 30min** (traz novos e alterados) + **full refresh 1x por dia de madrugada** (limpa tudo e reimporta, pegando os deletados). Perguntar ao cliente se o ERP tem algum campo de "data de exclusão" ou flag de inativo — facilita muito.

### Regras críticas

- **Nunca rodar o mesmo Data Loader em paralelo** — a tabela IMP_ é limpa a cada execução. Rodar duas vezes ao mesmo tempo causa perda de dados.
- **Data Loaders diferentes podem rodar em paralelo** — cada um tem sua própria tabela IMP_.
- **Data Loader e upsert devem ser SFs separadas** — cada uma tem seu próprio limite de tempo (300 segundos).

---

## 8. Importação de CSV

### Quando usar

Quando o dado não vem de um banco ou API, mas de um **arquivo que o usuário sobe manualmente** (planilhas de orçamento, metas, planejamento).

### Como funciona (2 etapas)

São duas peças que trabalham juntas:

**Etapa 1 — Upload (no frontend, na tela do usuário):**

O usuário escolhe o arquivo CSV na tela e o frontend faz o upload usando `uploadFileLoadableMitra`. "Loadable" é uma pasta especial do projeto onde ficam os arquivos que podem ser lidos pelo comando `LOAD DATA` do banco. Sem esse upload, a SF não consegue encontrar o arquivo.

```typescript
import { uploadFileLoadableMitra, executeServerFunctionMitra } from 'mitra-interactions-sdk';

// 1. Usuário clicou em "Importar" e selecionou o arquivo
await uploadFileLoadableMitra({ projectId: 12345, file: arquivoDoUsuario });

// 2. Agora chama a SF que processa o arquivo
await executeServerFunctionMitra({
  projectId: 12345,
  serverFunctionId: 30,  // SF tipo SQL com LOAD DATA
  input: { mes: 3, ano: 2025, cr: 'CR001' }
});
```

> A IA monta essa tela com um botão de upload + campos para o usuário preencher (mês, ano, CR, etc.)

**Etapa 2 — Processamento (SF tipo SQL com LOAD DATA):**

A SF tipo SQL lê o arquivo que foi enviado e insere os dados na tabela:

```sql
-- 1. Limpar dados antigos daquele segmento
DELETE FROM ORCAMENTO WHERE MES = {{mes}} AND ANO = {{ano}} AND CR = '{{cr}}';

-- 2. Carregar dados do CSV para a tabela
LOAD DATA LOCAL INFILE '${LOADABLE_FILE:arquivo.csv}'
INTO TABLE ORCAMENTO
CHARACTER SET utf8
FIELDS TERMINATED BY ','      -- separador: vírgula
ENCLOSED BY '"'               -- texto entre aspas
LINES TERMINATED BY '\n'      -- quebra de linha
IGNORE 1 LINES               -- pular cabeçalho
(CONTA, DESCRICAO, VALOR_PLANEJADO, VALOR_REALIZADO);

-- 3. Preencher colunas de contexto (vêm do input, não do CSV)
UPDATE ORCAMENTO SET MES = {{mes}}, ANO = {{ano}}, CR = '{{cr}}'
WHERE MES IS NULL;
```

**Fluxo visual completo:**
```
Gestor abre a tela → Seleciona "Março 2025" e "CR001" → Escolhe o CSV → Clica "Importar"
  │
  ├── Frontend faz upload do arquivo (uploadFileLoadableMitra)
  │
  ├── Frontend chama SF SQL (executeServerFunctionMitra)
  │     │
  │     ├── DELETE dados antigos de Mar/2025 + CR001
  │     ├── LOAD DATA insere linhas novas do CSV
  │     └── UPDATE preenche MES, ANO, CR nos registros novos
  │
  └── Tela mostra "Importação concluída!" ✅
```

### Exemplo: orçamento mensal por centro de resultado

Cada gestor sobe um CSV por mês com os dados do seu CR. O novo arquivo substitui apenas os dados daquele CR naquele mês — os dados de outros gestores ficam intactos.

### O que perguntar ao cliente

- Quem vai fazer o upload? (gestor na tela ou processo automatizado?)
- Qual a frequência? (mensal, semanal, diário?)
- O CSV novo substitui ou acumula dados?
- Tem alguma segmentação? (cada pessoa importa só seus dados?)
- Qual o encoding e separador? (UTF-8? vírgula? ponto-e-vírgula?)

---

## 9. Cron — agendamento automático

O cron é um agendamento que é **configurado numa Server Function**. Ele define **quando** aquela SF roda automaticamente, sem ninguém precisar disparar. Qualquer SF (SQL, INTEGRATION ou JAVASCRIPT) pode ter um cron.

| Exemplo | Significado |
|---------|-------------|
| A cada 30 minutos (horário comercial) | Dados atualizados durante o dia |
| Diário às 3h da manhã | Full refresh de madrugada quando ninguém usa |
| A cada 5 minutos | Mínimo permitido — para dados quase real-time |

### Sugestão padrão

- **Incremental** (só novidades): a cada 30min durante horário comercial
- **Full refresh** (tudo): 1x por dia de madrugada
- Sempre perguntar ao cliente se esse padrão atende

---

## 10. Tela de gerenciamento de integrações

Toda integração **deve ter uma tela de monitoramento** onde o cliente vê:

- **Status de cada entidade:** última importação, sucesso/erro, há quanto tempo
- **Logs de execução:** quanto tempo levou, quantos registros importados, erros detalhados
- **Botão "Executar Agora":** para rodar importação manualmente
- **Alertas por e-mail:** notificar quando uma integração falha
- **Visualização da tabela:** ver os dados importados em tempo real

> **Regra obrigatória:** Todo dashboard que consome dados importados deve mostrar **"Última atualização: há X minutos"** visível ao usuário.

---

## 11. Boas práticas — resumo

| Regra | Por quê |
|-------|---------|
| Sempre preferir SF SQL > INTEGRATION > JAVASCRIPT | SQL é 250x mais rápido que JavaScript |
| Usar Tabelas Online para centralizar JOINs | Evita repetir a mesma query em 5 SFs diferentes |
| Importar em lotes (por mês, por filial) | Não travar o ERP com queries gigantescas |
| Nunca rodar o mesmo Data Loader em paralelo | A tabela temporária é limpa — dados se perdem |
| DL e upsert em SFs separadas | Cada um precisa dos seus 300s de timeout |
| Nunca deletar + inserir sem proteção | Usuário pode ver dados vazios entre as operações |
| Sempre ter tela de logs | O cliente precisa saber se a importação está funcionando |
| Mostrar "Última atualização" no dashboard | O cliente precisa saber se os dados estão frescos |
| Alinhar TUDO com o cliente antes de codar | Frequência, modelo, deletados, queries — tudo |

---

## 12. Como pedir para a IA

Alguns exemplos de como orientar a IA:

| Você diz | A IA entende |
|----------|-------------|
| "Conecta no banco Oracle do cliente" | Criar JDBC (e sugerir Tunnel se on-premise) |
| "Integra com a API do Sankhya" | Criar Integração com template + SF INTEGRATION |
| "Quero importar pedidos do ERP a cada 30 minutos" | Data Loader + SF de orquestração + cron |
| "Os gestores vão subir CSV de orçamento todo mês" | Tela de upload + SF SQL com LOAD DATA |
| "Quero ver os dados em tempo real" | SFs apontando direto pro JDBC externo |
| "Cria uma tela pra acompanhar as importações" | Tela de gerenciamento com logs e status |
| "Os dados do dash estão desatualizados" | Revisar cron, logs, erros de importação |
| "Usa Tabela Online pra otimizar" | Centralizar JOINs + SFs referenciando @VW_ |
