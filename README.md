# 02 - Monitor de Emendas Brasil ⚙️ (Engenharia de Dados & Orquestração)

Este repositório contém a arquitetura de backend e o pipeline de dados do projeto **Monitor de Emendas Brasil**. Construído utilizando a plataforma **n8n**, este microsserviço atua como o motor central de orquestração, responsável pelos processos de Extração, Transformação e Carga (ETL), além da integração com modelos de Inteligência Artificial para análise contínua dos repasses governamentais.

👉 **Veja a Interface de Usuário (Front-end) aqui:** [Repositório 01 - Monitor de Emendas (UI)](https://github.com/CarloGiacomoni/01-monitor_de_emendas_brasil-Lovable)

---

## 🏗️ Estado Atual: Desacoplamento e Processamento Assíncrono (V2)

Para suportar o tempo de processamento inerente aos Modelos de Linguagem (LLMs) e evitar erros de *timeout* no front-end, a arquitetura atual é assíncrona e orientada a eventos, dividida em dois microsserviços interligados:

### 1. Fluxo de Recepção ("O Porteiro" / API Gateway)
Responsável por garantir alta disponibilidade e resposta em milissegundos para a interface de usuário (UI).
* **Ingestão (Webhook):** Recebe o payload via POST do front-end.
* **Controle de Estado:** Registra imediatamente a solicitação no banco de dados (Supabase) com status pendente.
* **Gatilho Assíncrono:** Invoca o "Fluxo Cérebro" em *background* sem aguardar sua conclusão.
* **Liberação:** Retorna um status HTTP 200 OK para o front-end imediatamente.

<img width="1649" height="774" alt="Fluxo_v4_p" src="https://github.com/user-attachments/assets/70a74e77-c345-4399-a8f3-c50c561e1386" />


### 2. Fluxo de Processamento Cognitivo ("O Cérebro" / Data Worker)
Motor pesado de ETL e inteligência artificial que roda em segundo plano.
* **Extração e Limpeza:** Recupera os metadados brutos do banco de dados e aplica higienização via código customizado.
* **Enriquecimento de Contexto:** Consulta feeds RSS externos em tempo real, limitando e agregando informações para alimentar a IA com notícias recentes.
* **Processamento de IA:** Orquestra o modelo Google Gemini Chat com injeção de memória para análise investigativa e geração de resumos em linguagem cidadã.
* **Pós-Processamento e Callback:** Higieniza o output bruto do LLM (removendo marcações indesejadas) e executa o *Update* no banco de dados, sinalizando a conclusão para o front-end.

<img width="1668" height="772" alt="Fluxo_v4_c" src="https://github.com/user-attachments/assets/4bc75d74-7b40-4f23-9281-02b2446ab86c" />


---

## 🚀 Evolução Arquitetural e Desafios Superados (V1 para V2)

A arquitetura atual é fruto de iterações contínuas para superar limites de infraestrutura e comportamento de IA.

### 1. Superando Limites de Hardware (OOM Killer)
Nas fases iniciais (V1), o maior desafio técnico foi rodar o pipeline massivo em uma VPS do Google Cloud com apenas **1 GB de Memória RAM**. 
* **A Solução:** A base central de dados (~295.000 linhas) foi fatiada em 1.652 arquivos CSV individuais no OneDrive. O n8n baixava apenas a fatia exata requisitada, protegendo a RAM. A VPS também recebeu 2 GB de memória Swap. *Na V2, essa barreira foi superada migrando os dados para o Supabase (PostgreSQL em nuvem).*

### 2. Resiliência Cognitiva: O "Airbag" em JavaScript
Para contornar a natureza probabilística das LLMs (que ocasionalmente ignoram prompts e respondem com texto livre em vez do JSON obrigatório), desenvolvi um "Airbag Lógico" via nó de JavaScript. Ele atua como extrator cirúrgico e contingência à prova de falhas:

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
```


---

## 📂 Como Importar os Fluxos
Este repositório contém os arquivos JSON dos fluxos. Importe-os diretamente para o seu editor n8n para replicar a arquitetura. É necessário configurar as credenciais do Supabase e a API Key do Google Gemini.
