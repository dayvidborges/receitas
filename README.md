# Documentação Completa do Sistema de Processamento de Imagens para Prescrições

**Data de escrita:** 2025-02-15  
**Autor:** Dayvid Borges

---

## 1. Visão Geral do Sistema

Este sistema é responsável pelo processamento assíncrono de imagens de documentos, com foco na validação e extração de informações de prescrições médicas e encaminhamentos para exames. A arquitetura envolve uma aplicação Django que integra o Celery para o processamento em background, além da comunicação com um serviço de inteligência artificial (IA) que analisa a imagem e retorna informações estruturadas.

## 1.1 SETUP DOS CONTAINERS

```bash
docker build
docker compose up
```

---

## 2. Arquitetura e Componentes

- **Django Backend:**  
  Responsável por receber as requisições do usuário na porta `localhost:8777`, armazenar os dados da tarefa e disparar o processamento.

- **Celery Worker:**  
  Responsável por executar a task `process_data`, que lê o arquivo de imagem, envia para o serviço de IA e processa a resposta, atualizando o status da tarefa e enviando um webhook de callback.

- **Serviço de IA:**  
  Disponível em `http://localhost:13111/analisar-prescricao`, este serviço analisa a imagem enviada e retorna um JSON com os dados de interpretação, como tipo de documento, informações do paciente, data de emissão e status da assinatura.

- **Webhook:**  
  Após o processamento, o sistema realiza uma chamada POST para a URL de webhook fornecida pelo usuário, enviando o resultado final da tarefa.

---

## 3. Modelo de Dados

### ProcessingTask

A classe `ProcessingTask` é um modelo Django que armazena as informações necessárias para o processamento. Os principais atributos são:

- **document_type (str):**  
  Tipo de documento informado pelo usuário (por exemplo, "prescription", "Exam referral", etc.).

- **external_reference (str):**  
  Referência externa fornecida pelo usuário para identificar a tarefa.

- **reference (UUID):**  
  Identificador único da tarefa, gerado automaticamente.

- **input_data (FileField):**  
  Arquivo de imagem enviado pelo usuário.

- **status (str):**  
  Estado atual da tarefa, podendo ser "pending", "processing", "completed" ou "failed".

- **result (JSONField):**  
  Resultado do processamento, contendo os dados retornados pela IA ou a mensagem de erro.

- **webhook_url (URLField):**  
  URL para a qual o resultado do processamento será enviado via POST.

- **created_at / updated_at (DateTimeField):**  
  Datas de criação e atualização da tarefa.

---

## 4. Fluxo de Processamento

1. **Recepção da Requisição:**  
   O usuário envia uma requisição para `localhost:8777`, incluindo o arquivo de imagem, o `document_type`, o `external_reference` e a `webhook_url`.

2. **Criação da Tarefa:**  
   Uma instância de `ProcessingTask` é criada com status inicial "pending".

3. **Execução da Task (`process_data`):**  
   - A task é alterada para o status "processing".
   - O arquivo de imagem é lido a partir do campo `input_data`.
   - Uma requisição POST é enviada para o endpoint do serviço de IA em `http://localhost:13111/analisar-prescricao`, com:
     - **Files:** O arquivo de imagem (chave `file`).
     - **Data:** Um dicionário contendo o `document_type`.

4. **Processamento da Resposta da IA:**  
   - A resposta JSON é processada e campos como `status`, `result`, `interpreter`, `cost` e `threads` são extraídos.
   - Caso o `status` seja "success", são aplicadas regras específicas:
     - **Documento Válido:** Se o `document_type` retornado for "prescription" ou "Exam referral", os campos extraídos do `result` (como `patient_name`, `issue_date`, `signature_status` e `accuracy`) são incluídos no payload.
     - **Documento Inválido ou Ilegível:** Se o resultado indicar que o documento não é válido ou não é legível, o payload conterá campos como `is_valid`, `is_legible` e uma mensagem explicativa.
   - Se o `status` não for "success", o payload será composto com os detalhes do erro.

5. **Envio do Webhook:**  
   O payload final é enviado via POST para a URL de webhook especificada no `ProcessingTask`.

6. **Atualização da Tarefa:**  
   O status da tarefa é atualizado para "completed" ou "failed" dependendo do sucesso do processamento.

---

## 5. Integração com o Serviço de IA

- **Endpoint:**  
  `http://localhost:13111/analisar-prescricao`

- **Parâmetros Enviados:**
  - **Arquivo:** Enviado em formato binário na chave `file`.
  - **document_type:** Enviado no corpo da requisição (form data).

- **Resposta Esperada:**  
  O serviço de IA retorna um JSON que pode conter as seguintes chaves:
  - `status`: Indicador do sucesso ou falha do processamento.
  - `result`: Dados do documento processado, podendo ser interpretados com `eval()` para extrair campos específicos.
  - `interpreter`: Dicionário contendo uma chave `content`, a partir do qual o `document_type` real é verificado.
  - `cost`: Custo ou tempo de processamento.
  - `threads`: Lista de threads/processos usados, que também são avaliados com `eval()`.

---

## 6. Payloads de Entrada e Saída

### 6.1 Payload de Entrada (para criação da tarefa)

- **Requisição do usuário (para o endpoint `localhost:8777`):**
  - **input_data:** Arquivo de imagem (arquivo binário).
  - **document_type:** Tipo de documento informado (ex.: "prescription" ou "Exam referral").
  - **external_reference:** Referência externa fornecida pelo usuário.
  - **webhook_url:** URL para onde será enviado o resultado do processamento.

- **Payload enviado ao serviço de IA:**
  - **Files:**  
    - `file`: O arquivo de imagem.
  - **Data (form data):**  
    - `"document_type": <tipo_de_documento>`

### 6.2 Payloads de Saída (enviados ao webhook)

#### Caso 1: Documento Válido (Prescrição ou Encaminhamento de Exame)

```json
{
  "document_type": "prescription",
  "status": "success",
  "patient_name": "Nome do Paciente",
  "issue_date": "DD/MM/YYYY",
  "signature_status": "Digital/Manual/Not signed",
  "accuracy": 90,
  "external_reference": "ref_exemplo",
  "reference": "uuid-string",
  "created_at": "2025-02-15 10:30:00",
  "ref": "ref_exemplo"
}
```

#### Caso 2: Documento Inválido ou Ilegível

```json
{
  "document_type": "documento_invalido",
  "accuracy": 75,
  "status": "invalid",
  "is_valid": false,
  "is_legible": false,
  "message": "Documento inválido.",
  "external_reference": "ref_exemplo",
  "reference": "uuid-string",
  "created_at": "2025-02-15 10:30:00",
  "ref": "ref_exemplo"
}
```

#### Caso 3: Falha no Processamento (status diferente de "success")

```json
{
  "document_type": "prescription",
  "status": "failed",
  "result": "<detalhes do erro ou resposta>",
  "external_reference": "ref_exemplo",
  "reference": "uuid-string",
  "created_at": "2025-02-15 10:30:00",
  "ref": "ref_exemplo"
}
```

---

## 7. Exemplo de Fluxo Completo

1. **Início:**  
   O usuário envia uma requisição para `localhost:8777` com o arquivo da imagem, o `document_type`, o `external_reference` e a `webhook_url`.

2. **Criação da Tarefa:**  
   Uma nova instância de `ProcessingTask` é criada e armazenada no banco de dados com status "pending".

3. **Processamento:**  
   O Celery worker executa a task `process_data`, atualizando o status para "processing" e enviando o arquivo para o serviço de IA.

4. **Resposta da IA:**  
   - Se o processamento for bem-sucedido (`status: success`) e o documento for válido (tipo "prescription" ou "Exam referral"), os dados extraídos (como nome do paciente, data de emissão, status da assinatura e acurácia) são incluídos no payload.
   - Se o documento for inválido ou ilegível, o payload conterá campos específicos indicando essa condição, como `is_valid`, `is_legible` e uma mensagem.

5. **Webhook:**  
   O payload final é enviado via POST para a `webhook_url` fornecida, notificando o usuário com o resultado do processamento.

6. **Finalização:**  
   A tarefa é marcada como "completed" em caso de sucesso ou "failed" em caso de erro, e os detalhes são armazenados no campo `result`.

---
