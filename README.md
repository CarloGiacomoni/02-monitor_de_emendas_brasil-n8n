# 02 - Emenda Insight IA ⚙️ (Backend & Automação)

Este repositório contém o "motor" de processamento e orquestração do projeto **Emenda Insight IA**. Ele é o microsserviço backend responsável por conectar o front-end, o repositório de dados fatiados (Data Lake no OneDrive) e a API do Google Gemini, rodando sob severas restrições de infraestrutura em nuvem.

👉 **Veja a Interface de Usuário (Front-end) aqui:** [Repositório 01 - Front-end Lovable]([LINK_PARA_O_SEU_REPOSITORIO_01_AQUI](https://github.com/CarloGiacomoni/01-emenda-insight-ia-Lovable)

## 🏗️ Visão Geral da Arquitetura (n8n)

O backend foi projetado sob os princípios de microsserviços. O fluxo foi desenhado de forma "stackable" (empilhada) no n8n para isolar as fases lógicas da esteira de dados:

![Arquitetura empilhada do n8n](Fluxo-n8n.jpeg)

* **Linha 1 — Ingestão:** Recebimento da requisição via Webhook e varredura de metadados no Data Lake (OneDrive).
* **Linha 2 — ETL:** Isolamento cirúrgico de **estritamente 1 arquivo CSV** via Filtro Inteligente (ignorando extensões e caracteres especiais via `contains`) e processamento de download.
* **Linha 3 — Inteligência & Output:** Processamento Cognitivo (Google Gemini 1.5), higienização de saída via JavaScript e devolutiva estruturada ao front-end.

## 🛡️ O Desafio de Infraestrutura: Superando o OOM Killer

O maior desafio técnico deste projeto foi rodar um pipeline de IA e processamento de dados massivos em uma VPS do Google Cloud com apenas **1 GB de Memória RAM**. A estratégia para evitar o travamento do servidor Linux (*Out of Memory Killer*) baseou-se em:

1.  **Fatiamento Granular (Python):** A base central (~60MB / 295.000 linhas) foi fatiada em 1.652 arquivos CSV individuais (um por parlamentar). O n8n baixa apenas a fatia exata requisitada (poucos KB), protegendo a RAM.
2.  **Memória Virtual:** A VPS foi configurada com 2 GB de memória Swap para absorver picos de processamento do Docker com segurança.

## 🛟 Lógica de Contingência: O "Airbag" em JavaScript

Para contornar a natureza probabilística das LLMs (que ocasionalmente ignoram prompts e respondem com texto livre em vez do JSON obrigatório), implementou-se um "Airbag Lógico" no nó de JavaScript. 

Ele atua como um extrator cirúrgico e um mecanismo de contingência à prova de falhas:

```javascript
// Captura a saída bruta gerada pela IA
let textoCru = $input.first().json.output || "";
textoCru = textoCru.replace(/```json/gi, '').replace(/```/g, '').trim();

// Localiza cirurgicamente apenas o objeto JSON
let inicioJson = textoCru.indexOf('{');
let fimJson = textoCru.lastIndexOf('}');
if (inicioJson !== -1 && fimJson !== -1) {
    textoCru = textoCru.substring(inicioJson, fimJson + 1);
}

let objetoJson;
try {
    objetoJson = JSON.parse(textoCru);
} catch (erro) {
    // MODO DE EMERGÊNCIA (Airbag): Envelopa o texto livre na estrutura do front-end
    objetoJson = { 
        "resposta_chat": "Processamento em contingência. Análise: \n\n" + textoCru,
        "anomalia": "Dados não estruturados nativamente.",
        "insight": "Processamento de contingência ativado.",
        "monitorando": "Acompanhando."
    };
}
return [{ json: objetoJson }];


📂 Como Importar o Fluxo
Este repositório contém o arquivo fluxo_n8n_emenda_insight.json. Importe-o diretamente para o seu editor n8n para replicar a arquitetura. É necessário reconfigurar as credenciais OAuth2 do Microsoft OneDrive e a API Key do Google Gemini.
