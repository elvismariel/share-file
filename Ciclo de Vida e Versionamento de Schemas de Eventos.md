Este é um rascunho de documentação técnica voltado para usuários da plataforma de eventos (desenvolvedores, arquitetos de integração e engenheiros de dados). O documento foi estruturado com base nas premissas fornecidas: uso de **"Backward Compatibility"** (Compatibilidade Reversa) e **versionamento numérico crescente no caminho do arquivo**.

# ---

**Guia de Ciclo de Vida e Versionamento de Schemas de Eventos**

```mindmap
  root((Plataforma de Eventos))
    Governança
      Ciclo de Vida
        Draft
        Active
        Deprecated
        Retired
      Versionamento
        Major (Quebrante)
        Minor (Evolução)
        Path-based (Ex: /v2/)
    Contratos (Canônico)
      Schema Registry
      Backward Compatibility
      Validação de Dados
    Atores
      Produtores (Emitem)
      Consumidores (Reagem)
      Tópicos/Bus (Transportam)
```

## **1\. Introdução**

Este guia destina-se a todos os produtores e consumidores de eventos que utilizam a nossa plataforma de mensageria assíncrona. O objetivo é estabelecer um padrão claro para a evolução dos contratos de dados (schemas), garantindo a estabilidade operacional e o desacoplamento real entre microsserviços.

A governança dos nossos eventos é sustentada por dois pilares fundamentais:

1. **Padrão de Compatibilidade Reversa (Backward Compatibility).**  
2. **Versionamento Numérico Crescente no Caminho (Path-based Versioning).**

## **2\. Conceitos-Chave**

## **2.1. O que é "Backward Compatibility"?**

Um schema é considerado **reversamente compatível** quando os consumidores que utilizam a nova versão do schema ainda conseguem ler e processar os dados produzidos pela versão anterior.

**Regra de Ouro:** Novas versões do schema **não podem quebrar** consumidores antigos que operam na mesma versão principal (major version, definida pelo caminho do arquivo).

Para garantir isso, as regras de evolução são estritas:

* **Permitido:** Adicionar campos opcionais (sem valor padrão obrigatório).  
* **Permitido:** Remover campos opcionais (com tratamento no consumidor).  
* **Proibido:** Adicionar campos obrigatórios.  
* **Proibido:** Remover campos obrigatórios.  
* **Proibido:** Alterar o tipo de dado de um campo existente (ex: mudar de string para integer).  
* **Proibido:** Renomear campos existentes.

## **2.2. Versionamento no Caminho (Path-based Versioning)**

Ao contrário de sistemas que usam revisões implícitas em um banco de dados, o nosso **Schema Registry** reflete diretamente a estrutura de diretórios do repositório de schemas (como o Bitbucket/GitLab). A versão principal do evento é um número no caminho do diretório.

**Como identificar a versão:**

Observe a imagem abaixo, extraída do nosso repositório canonical/foreign-exchange:

\[image\_0.png\]

O caminho do arquivo destacado é:

foreign-exchange / forex-aml-service / **2** / foreign-exchange.forex-aml-service.contract.settled.json

O número **2** no caminho do diretório indica a **versão principal (Major Version)** do contrato.

## ---

**3\. Os Ciclos de Vida das Versões de Eventos**

Uma versão principal de um evento passa por estados específicos para garantir a previsibilidade para os consumidores.

\[Image conceptual: Flow diagram showing states: Proposed \-\> Active (vX) \-\> Deprecated (vX-1) \-\> Retired (vX-2)\]

| Estado | Descrição | Regras de Governança |
| :---- | :---- | :---- |
| **1\. Proposed (Em Proposta)** | O schema está sendo desenvolvido em uma *branch* de *Pull Request* no Git. | Não pode ser usado em produção. Os consumidores devem usá-lo apenas para testes de integração e validação inicial. |
| **2\. Active (Ativo)** | A versão atual da verdade. É a versão que todos os produtores devem emitir. | Corresponde ao arquivo na branch master/main. O Schema Registry valida todos os eventos de produção contra esta versão. É o estado de .../v2/... na imagem de exemplo. |
| **3\. Deprecated (Depreciado)** | Uma versão anterior (ex: v1) ainda é suportada, mas não é recomendada para novas integrações. | Possui uma data de **"Sunset" (Fim de Vida)** definida. Novos consumidores devem obrigatoriamente usar a versão Active. Os produtores são notificados para migrar para a versão Active antes da data de Sunset. |
| **4\. Retired (Aposentado)** | O schema foi removido da plataforma e não é mais suportado pelo Schema Registry. | Eventos enviados usando esta versão falharão na validação. O diretório correspondente foi removido da branch master e arquivado. |

## ---

**4\. Guia Prático de Evolução de Schemas**

Esta seção detalha o fluxo de trabalho passo a passo usando o exemplo prático fornecido.

**Ponto de Partida (Estado Atual em Produção):**

O contrato settled da versão 2, conforme o arquivo JSON fornecido:

JSON

// foreign-exchange / forex-aml-service / 2 / ...contract.settled.json  
{  
  "type":"object",  
  "$schema": "https://json-schema.org/draft-07/schema\#",  
  "definitions": {},  
  "title": "foreign-exchange.fore-aml-service.contract.settled",  
  "description": "Message posted on the platform indicating the ocurrence of the settlement of an exchange contract",  
  "required": \[  
    "event\_type",  
    "reference\_number"  
  \],  
  "properties": {  
    "event\_type": { "type": "string", "description": "Exchange event type" },  
    "reference\_number": { "type": "string", "description": "Exchange contract number" },  
    "bacen\_control\_number": { "type": "string", "description": "Bacen control number" }  
  }  
}

Temos 3 campos principais: event\_type (obrigatório), reference\_number (obrigatório) e bacen\_control\_number (opcional).

## **Cenário 1: Evolução Não-Quebrante (Backward Compatible)**

*Desejamos adicionar um novo campo chamado settlement\_country (opcional).*

**Fluxo de Trabalho:**

1. **Criação da Proposta:** O time responsável pelo forex-aml-service cria uma nova branch de PR.  
2. **Edição *No-Quebrante*:** Eles editam o arquivo **dentro da pasta 2** (conforme a imagem). Eles **não** mudam o diretório para 3\.  
3. **Atualização do JSON (v2-Revisão):**  
   JSON  
   // foreign-exchange / forex-aml-service / 2 / ...contract.settled.json  
   "required": \[  
       "event\_type",  
       "reference\_number"  
       // NOTA: NÃO ADICIONAMOS O NOVO CAMPO AQUI, POIS ELE É OPCIONAL  
   \],  
   "properties": {  
       // ...campos existentes...  
       "settlement\_country": { // NOVO CAMPO OPCIONAL  
         "type": "string",  
         "description": "ISO country code of the settlement"  
       }  
   }

4. **Aprovação:** O PR é revisado pelo time de governança de dados. Como é uma mudança não-quebrante (adicionando um campo opcional em um schema existente), ele é aprovado.  
5. **Merge:** O PR é mergeado para master.  
6. **Resultado no Ciclo de Vida:** O diretório /2/ permanece **Active**, mas o Schema Registry agora aceita eventos contendo o campo settlement\_country (e ainda aceita eventos sem ele).

## ---

**Cenário 2: Evolução Quebrante (Breaking Change)**

*Desejamos remover o campo obrigatório reference\_number (talvez substituindo-o por um ID universal) ou mudar o tipo de reference\_number de string para integer.*

**Fluxo de Trabalho:**

1. **Criação da Proposta:** O time responsável pelo forex-aml-service cria uma nova branch de PR.  
2. **Identificação de Breaking Change:** O time reconhece que remover um campo obrigatório quebra consumidores antigos que esperam esse campo.  
3. **Criação da Nova Versão:** Eles **não** modificam o arquivo na pasta 2\. Em vez disso, eles criam um **novo diretório** chamado 3 no caminho canonical/foreign-exchange/forex-aml-service/3/.  
4. **Criação do Novo JSON (v3):** Copiam o schema original da pasta 2 para a pasta 3 e aplicam as mudanças desejadas:  
   JSON  
   // foreign-exchange / forex-aml-service / 3 / ...contract.settled.json  
   {  
     "type":"object",  
     "$schema": "https://json-schema.org/draft-07/schema\#",  
     // ...metadados atualizados...  
     "required": \[  
       "event\_type"  
       // reference\_number FOI REMOVIDO DAQUI  
     \],  
     "properties": {  
       "event\_type": { "type": "string", "description": "Exchange event type" },  
       // reference\_number FOI TOTALMENTE REMOVIDO  
       "bacen\_control\_number": { "type": "string", "description": "Bacen control number" }  
     }  
   }

5. **Aprovação e Definição de Sunset:** O PR é revisado. A governança aprova a criação da V3. Simultaneamente, o time de governança **atualiza o status da V2 (pasta /2/)** no Schema Registry para **Deprecated**, definindo uma data de fim de vida (Sunset date).  
6. **Merge:** O PR é mergeado para master.  
7. **Resultado no Ciclo de Vida:**  
   * **Versão 3 (pasta /3/):** Torna-se a versão **Active**. Todos os novos projetos devem usá-la.  
   * **Versão 2 (pasta /2/):** Torna-se a versão **Deprecated**. Os produtores existentes continuam enviando eventos v2, mas são notificados para planejar a migração para a V3 antes da data de Sunset.

## ---

**Cenário 3: Aposentadoria de Versão (Sunset e Retirement)**

*A data de Sunset para a V2 chegou.*

**Fluxo de Trabalho:**

1. **Monitoramento:** O time de governança verifica que nenhum produtor está mais emitindo eventos v2. Se ainda houver, a migração é forçada.  
2. **Aposentadoria:** O time de governança remove o diretório /2/ da branch master e o move para um arquivo de histórico.  
3. **Resultado no Ciclo de Vida:** A versão 2 é marcada como **Retired** na plataforma. Qualquer tentativa de enviar um evento validado contra a pasta /2/ falhará.

## **5\. Resumo das Regras de Governança**

* **Validação Obrigatória:** Todos os eventos de produção **devem** ser validados contra o schema no Schema Registry antes de serem publicados.  
* **Tratamento de Campos Desconhecidos:** Os consumidores devem ser implementados para **ignorar** campos que eles não reconhecem no evento, garantindo que adições não-quebrantes não os derrubem.  
* **Versionamento numérico no caminho:** Use o número do diretório no caminho do arquivo para sinalizar breaking changes e gerenciar o ciclo de vida. Mantenha os arquivos dentro do mesmo diretório para evoluções não-quebrantes.