# Manual de Implementação e Uso: Colaboração - Flowgrammers

**Fornecido por:** Gabriel Siqueira

Este manual descreve a operação, configuração e lógica dos workflows de automação para análise de tráfego pago (Facebook Ads) e criativos. O sistema é composto por 3 workflows interconectados que extraem métricas, classificam o desempenho e realizam análises qualitativas utilizando Inteligência Artificial.

## 📋 Visão Geral do Sistema

O sistema opera em três estágios principais:

1.  **📊 Extração de Métricas**: Coleta dados diários e históricos das campanhas e anúncios.
2.  **📊 Ranking (Classificação)**: Analisa o desempenho acumulado (últimos 30 dias) e identifica outliers (positivos e negativos).
3.  **📊 Análise Detalhada**: Aprofunda a análise nos outliers identificados, transcrevendo vídeos e analisando imagens/copy com IA para entender os motivos do desempenho.

---

## 1. Workflow: Extração de Métricas
**Arquivo**: `workflows/1-extraction-metrics.json`

Este workflow é responsável pela ingestão de dados brutos do Facebook Ads para uma planilha Google Sheets, servindo como base de dados para dashboards.

### Gatilhos (Triggers)
*   **Agendamento**: Executa automaticamente todos os dias às **11:00** e **20:00**.
*   **Manual**: Pode ser acionado manualmente via `Execute Workflow Trigger` para testes.

### Fluxo de Processamento
1.  **Definição de Período**:
    *   O sistema determina se a execução é para dados "Pretéritos" (histórico do ano anterior até hoje) ou para a janela padrão de **8 dias** (hoje + 7 dias anteriores).
    *   *Nota*: A lógica de "Pretéritos" é ativada se uma variável de timestamp não estiver presente, gerando uma lista longa de datas.
2.  **Iteração por Contas**:
    *   Obtém a lista de contas de anúncio a serem processadas de uma planilha de controle (`Get: Contas`).
    *   Itera sobre cada conta (`SplitInBatches`).
3.  **Busca no Facebook (Fetch Metrics)**:
    *   Para cada data definida, consulta a API `insights` do Facebook (nível `ad`).
    *   Campos buscados: `spend`, `impressions`, `reach`, `actions`, `action_values`, métricas de vídeo (`video_p***`), etc.
4.  **Tratamento de Dados (Javascript)**:
    *   O nó `Extrai Metricas da Array Actions` converte os arrays complexos do Facebook (ex: `actions: [{action_type: 'link_click', value: 10}]`) em colunas planas (`actions_count_link_click: 10`).
5.  **Persistência**:
    *   Os dados tratados são enviados para a planilha Google Sheets (`Adiciona à Planilha`), aba `dados`.

### Inputs e Outputs
*   **Input**: ID da Planilha mestre (via nó `Variables`).
*   **Output**: Linhas adicionadas na planilha Google Sheets com métricas granulares por Anúncio/Dia.

---

## 2. Workflow: Ranking de Criativos (Classificação)
**Arquivo**: `workflows/2-ranking-classification.json`

Este workflow avalia a qualidade dos criativos com base em dados agregados de 30 dias.

### Gatilhos
*   **Agendamento**: Executa periodicamente (configuração padrão: vazio/manual no arquivo, ajustar conforme necessidade).

### Fluxo de Processamento
1.  **Configuração de Parâmetros**:
    *   O nó `Data` define:
        *   `Date Preset`: `last_30d` (janela de análise).
        *   `Percentual Outliers Positivos`: `40%` (ex: 40% melhor que a média).
        *   `Percentual Outliers Negativos`: `30%` (ex: 30% pior que a média).
2.  **Busca de Dados Agregados**:
    *   Consulta a API do Facebook para os últimos 30 dias, trazendo métricas totais.
3.  **Cálculo Estatístico (Javascript)**:
    *   Calcula as médias da conta para: CPA, CPL, ROAS, CTR, CPC.
    *   **Lógica de Outlier Positivo** (`Outliers Positivos`):
        *   CPA ou CPL `X%` **abaixo** da média.
        *   ROAS ou CTR `X%` **acima** da média.
        *   Gera tags como: `🔥 CTR 50% acima da média`.
    *   **Lógica de Outlier Negativo** (`Outliers Negativos`):
        *   CPA ou CPL `X%` **acima** da média.
        *   ROAS ou CTR `X%` **abaixo** da média.
        *   Gera tags como: `⚠️ CPA 40% acima da média`.
4.  **Separação e Envio**:
    *   Criativos classificados como "Positivos" vão para a aba `Top`.
    *   Criativos classificados como "Negativos" vão para a aba `Piores`.

---

## 3. Workflow: Ranking - Análise Detalhada
**Arquivo**: `workflows/3-ranking-detailed-analysis.json`

Este é o workflow mais avançado, que enriquece os dados dos criativos classificados com análises qualitativas de IA.

### Gatilhos
*   **Agendamento**: Executa periodicamente.
*   **Dependência**: Lê as planilhas preenchidas pelo Workflow 2 (`Top` e `Piores`).

### Fluxo de Processamento
1.  **Leitura das Planilhas**:
    *   Lê as abas `Top` (Positivos) e `Piores` (Negativos) do Google Sheets.
    *   Filtra apenas linhas que ainda não foram processadas (coluna `checked` vazia ou diferente de ✅).
2.  **Obtenção do Criativo (Mídia)**:
    *   Usa o `ad_id` para buscar detalhes no Facebook (`Get: Detalhes do Anúncio`).
    *   **Identificação do Tipo de Mídia** (Switch `Tipo`):
        *   **Imagem**: Obtém URL via hash ou campo `image_url`.
        *   **Vídeo/Reels**: Obtém URL do vídeo.
        *   **Carrossel/Outros**: Trata conforme disponibilidade.
3.  **Processamento de Conteúdo**:
    *   **Vídeos**: Baixa o vídeo e envia para `Transcrever` (OpenAI Whisper) para obter o texto falado.
    *   **Imagens**: Envia a URL para `Análise de Imagem` (OpenAI GPT-4o-mini) com o prompt: *"Descreva esse criativo para que seja analisado posteriormente pelo estrategista"*.
4.  **Consolidação (Mapping)**:
    *   Unifica os dados: Título, Texto (Body), Descrição, Call to Action, Transcrição/Análise Visual, e Tipo do Criativo.
5.  **Atualização da Planilha**:
    *   Atualiza a linha correspondente na planilha (`Update Positivos` / `Update Negativos`) preenchendo as colunas de análise e marcando `checked` como ✅.

---

## 🛠 Guia de Configuração

Para implantar esses workflows em um novo ambiente n8n, siga estes passos:

### 1. Requisitos de Credenciais
Certifique-se de que as seguintes credenciais estejam configuradas no n8n:
*   **Facebook Graph API**: Permissões de leitura de `ads_read`, `read_insights`.
*   **Google Sheets OAuth2**: Permissão de leitura e escrita no Drive/Sheets.
*   **OpenAI API**: Chave de API válida com saldo para GPT-4o e Whisper.

### 2. Configuração de Variáveis (Hardcoded)
Alguns IDs são fixos nos nós e precisam ser alterados para cada cliente. 

**Onde alterar:**
*   **Workflow 1 (Extração)**:
    *   Nó `Variables`: Atualizar `ID Planilha Extração de Métricas` (se não for dinâmico).
    *   Nó `Get: Contas`: Verificar o ID da planilha de contas.
*   **Workflow 2 (Ranking)**:
    *   Nó `Data`:
        *   `ID Sheets Ranking de Criativos`: ID da planilha de destino.
        *   `Instância Evolution` / `URL Evolution`: Se houver integração com API de WhatsApp (Evolution API).
*   **Workflow 3 (Análise)**:
    *   Nó `Data`: Mesmo ID da planilha de Ranking.
    *   Nó `Data`: Variáveis de URL e parâmetros de integração.

### 3. Planilhas Google
As planilhas devem seguir a estrutura de colunas esperada pelos nós `Google Sheets`.
*   **Planilha de Contas**: Deve conter IDs das contas de anúncio (`act_...`).
*   **Planilha de Métricas**: Colunas para cada métrica (`spend`, `clicks`, etc.).
*   **Planilha de Ranking**: Abas `Top` e `Piores` com colunas para `ad_id`, `desempenho`, `transcription`, etc.

## ⚠️ Pontos de Atenção e Manutenção
*   **Tokens do Facebook**: Tokens de acesso costumam expirar (geralmente a cada 60 dias). Monitore erros de autenticação.
*   **Limites da API**: O Workflow 1 pode atingir limites de taxa se houver muitas contas/anúncios. O uso de `SplitInBatches` e `Wait` ajuda a mitigar isso.
*   **OpenAI Costs**: O Workflow 3 consome créditos da OpenAI para visão e transcrição. Monitore o uso.
