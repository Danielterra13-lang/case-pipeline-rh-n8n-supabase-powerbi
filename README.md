# Pipeline de Dados de RH com N8N, Supabase e Power BI

**Tipo:** Automação de Dados / Business Intelligence
**Stack:** N8N (self-hosted via Docker), Supabase (PostgreSQL), Power BI, assistente de IA embutido

## O problema

Uma planilha de RH em Excel com cadastro de colaboradores, sem padronização. Na prática: cabeçalhos repetidos a cada período de contratação, blocos de dados separados por linhas em branco, e registros reais misturados com linhas de seção. Ao todo, 90 colaboradores cadastrados espalhados em 107 linhas, o que já é um sinal de quanto ruído estrutural existe numa planilha "só de olhar".

📌 <img width="980" height="372" alt="image" src="https://github.com/user-attachments/assets/fd589f7f-9bae-4e77-b351-22999838ab15" />


Esse tipo de planilha funciona pra consulta pontual, mas trava qualquer análise: não dá pra montar um gráfico de evolução de headcount, cruzar desligamento com escolaridade, ou responder "quantas mulheres temos no time de desenvolvimento" sem antes gastar um bom tempo limpando tudo manualmente, toda vez que alguém precisar de novo.

## Visão geral da solução

Um pipeline ponta a ponta com três camadas bem separadas, cada uma fazendo uma coisa só:

1. **N8N**: monitora a planilha, extrai e envia os dados tratados para o banco.
2. **Supabase (Postgres)**: guarda o dado limpo e modelado em tabelas dimensão.
3. **Power BI**: consome o banco e entrega o dashboard, incluindo um chatbot pra perguntas em linguagem natural.

Separar assim, em vez de fazer tudo dentro do Power BI (que também tem Power Query), tem uma vantagem direta: o dado tratado fica disponível num banco real, então qualquer outra ferramenta (outro dashboard, uma API, um relatório em Python) pode consumir a mesma base sem repetir a limpeza.

## Arquitetura do pipeline (N8N + Docker)

O N8N roda self-hosted, via Docker Desktop (`n8nio/n8n:latest`, porta 5678), em vez de usar o N8N Cloud. Isso dá controle total sobre execução e credenciais, sem limite de execuções, mas também significa que a responsabilidade por uptime, backup dos workflows e atualização do container é sua, não de um provedor.

📌 *Imagem: images/07-docker-desktop-n8n.png*

O workflow em si segue esse fluxo:

```
Google Drive Trigger (fileUpdated)
    └─> Download file
        └─> Extract from File (Extract From XLSX)
            └─> Code (tratamento/normalização)
                └─> Loop Over Items
                    ├─> Create a row (Supabase) ──done──┐
                    │        └── Success ────────────────┼─> volta pro loop
                    │        └── Error ──> Create file from text (log de erro no Drive) ─┘
                    └─> loop
```

📌 *Imagem: images/02-pipeline-n8n.png* (ou images/06-workflow-n8n-completo.png para a versão sem corte)

Dois pontos de design que valem destaque:

- **Gatilho por atualização de arquivo, não por agendamento**: qualquer edição na planilha do Drive já dispara o pipeline inteiro. Simples de configurar, mas repare no efeito colateral abaixo.
- **Tratamento de erro dentro do próprio workflow**: quando o node "Create a row" falha, o N8N não trava o loop inteiro, ele desvia pra um node "Create file from text" que grava o erro num arquivo de log no Drive e segue o processamento dos itens seguintes. Foi exatamente esse mecanismo que capturou o erro real abaixo.

### Um erro real, e o que ele revela

O log de erro gerado pelo próprio pipeline registrou:

```
duplicate key value violates unique constraint "colaboradores_pkey"
```

Isso acontece porque o gatilho reprocessa a planilha inteira a cada atualização, e o node "Create a row" faz um INSERT simples. Se o arquivo é atualizado de novo (uma correção, uma linha nova), o pipeline tenta reinserir também os registros que já existiam, e o Postgres corretamente rejeita a chave duplicada.

O fluxo de erro fez o trabalho que deveria fazer (não travou nada, e deixou rastro), mas a causa raiz é uma escolha de arquitetura que vale corrigir: trocar o INSERT por um **upsert** (o próprio node do Supabase no N8N tem essa opção, baseada em `ON CONFLICT`), usando o ID do colaborador como chave de conflito. Isso resolve o erro na origem, em vez de só logá-lo.

## Modelagem de dados (Supabase)

No Postgres, o dado chega na tabela `colaboradores` e é organizado em tabelas dimensão: `dim_cargo`, `dim_escolaridade`, `dim_genero`, `dim_setor`, `dim_tipo_desligamento`.

📌 *Imagem: images/03-supabase.png*

Ponto de atenção que vale registrar no case: as tabelas aparecem marcadas como **Unrestricted**, e o editor mostra o aviso **RLS disabled** (Row Level Security desligado). Pra um projeto de portfólio, sem problema, mas é o mesmo tipo de decisão que precisa mudar antes de qualquer uso em produção: com RLS desligado, se a chave pública (anon key) do Supabase for exposta em algum lugar, qualquer um com essa chave lê a tabela direto pela API REST automática do Supabase. A correção é simples: habilitar RLS e liberar acesso só via service role, que é o que o N8N já usa internamente pra escrever.

## Camada de visualização (Power BI)

📌 *Imagem: images/04-dashboard-powerbi.png*

O dashboard cobre o essencial de um painel de RH:

- KPIs no topo com comparativo ano a ano: colaboradores ativos, contratações, demissões e média de folha salarial.
- Série temporal de colaboradores ativos entre 2011 e 2019.
- Distribuição por setor, por estado (mapa do Brasil), por gênero e por escolaridade (gráfico radar).
- Filtros de mês e ano.

## Chatbot no Power BI

📌 *Imagem: images/05-chatbot-powerbi.png*

O mesmo tipo de assistente de IA usado no case de vendas aparece aqui também, plugado ao modelo de RH. No exemplo capturado no próprio dashboard, uma pergunta em linguagem natural sobre desligamentos em 2019 recebe de volta o número exato já calculado pelo modelo, sem o gestor precisar abrir filtro nenhum.

O padrão se repete entre os dois projetos, o que é um bom sinal: a combinação Power BI mais chatbot virou um componente reaproveitável, não um recurso isolado de um único dashboard.

## Resultados

Com o recorte de 2019: 70 colaboradores ativos (+11,11% vs. ano anterior), 11 contratações (-31,25%), 6 demissões (+100%) e média de folha salarial de R$ 3 mil (+11,62%). Distribuição de 62,22% homens e 37,78% mulheres, com desenvolvimento como o setor de maior headcount.

O ganho real é o mesmo do outro case: uma planilha que antes exigia limpeza manual toda vez vira um pipeline que atualiza sozinho, e o gestor de RH consegue tirar dúvida direto no dashboard em vez de pedir um recorte novo pra alguém de dados.

## Trade offs e próximos passos

- **INSERT sem upsert**: como já descrito, é a causa do erro de chave duplicada. Trocar por upsert com base no ID do colaborador resolve na raiz.
- **RLS desligado no Supabase**: adequado para portfólio, mas precisa ser corrigido antes de qualquer dado real de RH passar por ali, já que é dado sensível por natureza.
- **N8N self-hosted num Docker local**: ótimo para desenvolvimento e para manter controle total, mas significa que o pipeline só roda enquanto essa máquina estiver ligada e com o container ativo. Pra produção, o caminho natural é uma instância de N8N hospedada (VPS, ou N8N Cloud) com backup automático dos workflows.
- **Reprocessamento do arquivo inteiro a cada trigger**: funciona para uma base de 90 registros, mas não escala. Com uma planilha maior, vale mudar a lógica pra processar só as linhas novas ou alteradas, em vez do arquivo inteiro a cada disparo.

*Case construído a partir de um projeto próprio de automação de dados de RH, integrando N8N, Supabase e Power BI num fluxo ponta a ponta, incluindo o tratamento de um erro real encontrado durante a execução do pipeline.*
