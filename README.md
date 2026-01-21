# Colaboração - Flowgrammers

**Fornecido por:** Gabriel Siqueira

Este repositório contém workflows do **n8n** desenvolvidos para automatizar a extração de métricas do Facebook Ads, classificar o desempenho de criativos e realizar análises qualitativas utilizando Inteligência Artificial.

## 📂 Conteúdo

O diretório `@flowg-metric` está organizado da seguinte forma:

*   **`workflows/`**: Contém os arquivos JSON dos workflows.
    1.  `1-extraction-metrics.json`: Coleta dados diários de campanhas e anúncios.
    2.  `2-ranking-classification.json`: Classifica criativos em "Top" ou "Piores" baseado em KPIs.
    3.  `3-ranking-detailed-analysis.json`: Enriquece a análise transcrevendo vídeos e analisando imagens com IA.
*   **`docs/`**: Documentação detalhada.
    *   [Manual de Implementação e Uso](docs/MANUAL.md)

## 📖 Documentação Completa

Para um guia detalhado sobre como implementar, configurar e utilizar esses workflows, consulte o **[Manual de Implementação e Uso](docs/MANUAL.md)**.

The manual covers:
*   Logic behind each node.
*   Required dependencies and credentials.
*   Variables and Spreadsheet IDs that need configuration.
*   Structure of data spreadsheets.

## 🚀 Como Começar

1.  **Importe os Workflows**: Importe os arquivos `.json` da pasta `workflows/` para sua instância do n8n.
2.  **Configure as Credenciais**: Configure as credenciais do Facebook Graph API, Google Sheets e OpenAI.
3.  **Ajuste os IDs**: Siga o [Guia de Configuração no Manual](docs/MANUAL.md#%F0%9F%9B%A0-guia-de-configura%C3%A3o) para apontar para suas próprias planilhas (substituindo as variáveis `{{SHEET_ID...}}`).
4.  **Teste**: Execute os workflows manualmente para validar o funcionamento antes de ativar os agendamentos.

## ⚙️ Tecnologias Utilizadas
*   **n8n**: Orquestração de workflows.
*   **Facebook Graph API**: Fonte de dados de anúncios.
*   **Google Sheets**: Banco de dados e interface de visualização.
*   **OpenAI API (GPT-4o & Whisper)**: Inteligência para análise de imagem e transcrição de áudio.
