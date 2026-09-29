# Especificação - Document Management System

## 1. Objetivo

Entregar uma aplicação web que permita a usuários enviar, listar e baixar seus documentos, mantendo os arquivos no filesystem local e os metadados em memória.

## 2. Escopo

### Dentro do escopo

- Enviar um arquivo por requisição e associá-lo a um usuário.
- Listar os documentos associados ao usuário informado na requisição.
- Baixar um documento próprio pelo identificador.
- Usar o filesystem local em `backend/storage` para os arquivos e um repositório em memória para seus metadados.
- Oferecer uma interface React para upload, listagem e download, consumindo a API pelo prefixo `/api`.

### Fora do escopo

- Autenticação ou autorização de produção. O identificador enviado pelo cliente nesta fase é apenas um mecanismo de protótipo.
- Banco de dados ou persistência dos metadados entre reinicializações.
- Armazenamento externo, em nuvem ou em serviços de terceiros.
- Versionamento, exclusão, busca, paginação, compartilhamento ou edição de documentos.
- Allowlist de tipos de arquivo, antivírus, processamento, compressão ou preview de conteúdo.

## 3. Requisitos funcionais

| ID | Requisito | Critério de aceite |
| --- | --- | --- |
| RF-01 | O usuário pode enviar um documento. | `POST /upload` aceita um arquivo no campo `file` de `multipart/form-data`, exige `X-User-Id` e responde `201` com os metadados públicos criados. |
| RF-02 | O sistema rejeita uploads inválidos. | Rejeita identidade ausente/inválida, arquivo ausente ou vazio, mais de um arquivo, campo de arquivo diferente de `file`, multipart malformado e arquivo acima do limite; retorna erro JSON conforme a seção 6. |
| RF-03 | O sistema associa cada documento a um usuário. | O `owner` é obtido exclusivamente do header `X-User-Id`; nenhum campo de formulário pode substituir o valor. |
| RF-04 | O usuário pode listar seus documentos. | `GET /documents` retorna somente documentos cujo `owner` corresponde ao header `X-User-Id`, do mais novo para o mais antigo. |
| RF-05 | O usuário pode baixar um documento próprio pelo identificador. | `GET /documents/:id/download` transmite os bytes do arquivo sem expor o caminho físico. |
| RF-06 | O sistema protege a privacidade entre usuários. | Documento inexistente, sem metadados ou pertencente a outro usuário retorna a mesma resposta `404`, sem revelar sua existência ou proprietário. |
| RF-07 | O sistema preserva o nome original como metadado. | O nome original é devolvido na API e usado com segurança no nome de download; nunca é usado como nome ou caminho físico de armazenamento. |
| RF-08 | O sistema mantém o endpoint de saúde existente. | `GET /health` continua respondendo `200` com `{ "status": "ok" }`, sem exigir `X-User-Id`. |

## 4. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| RNF-01 | Os arquivos devem ser gravados somente no filesystem local em `backend/storage`, usando Multer com `diskStorage`. Não usar armazenamento remoto ou serviços de terceiros. |
| RNF-02 | Os metadados devem ser mantidos em memória nesta fase. Uma reinicialização perde os metadados; arquivos que permanecerem no disco não podem ser listados nem baixados após essa perda. |
| RNF-03 | O limite padrão de upload é 10 MiB (`10485760` bytes), configurável por `MAX_UPLOAD_SIZE_BYTES`. A configuração deve ser validada no início; valores ausentes usam o padrão e valores não positivos ou não inteiros impedem a inicialização. |
| RNF-04 | A porta HTTP é configurada por `PORT`, com padrão `3000`, conforme a configuração existente. |
| RNF-05 | A API não confia em nome, caminho ou tipo de conteúdo fornecidos pelo cliente para decidir o caminho físico. O nome armazenado deve ser gerado pelo servidor e não conter o nome original. |
| RNF-06 | O backend usa Node.js e CommonJS; o frontend usa React, Vite e JavaScript ESM, sem TypeScript nesta fase. |
| RNF-07 | O backend deve manter as responsabilidades nas camadas `routes`, `controllers`, `services` e `repositories`, com dependências fluindo para dentro. |
| RNF-08 | O header `X-User-Id` não constitui autenticação. A aplicação não deve ser considerada segura para acesso público ou multiusuário real sem autenticação e autorização confiáveis. |
| RNF-09 | Respostas de erro da API são JSON no formato comum definido na seção 6, exceto após o início de uma transmissão binária, quando headers já enviados impedem uma resposta JSON substituta. |

## 5. Modelo de dados

### Metadados públicos do documento

| Campo | Tipo | Descrição |
| --- | --- | --- |
| `id` | string | UUID v4 gerado pelo servidor, usado nas rotas da API. |
| `originalName` | string | Nome original recebido do cliente, preservado apenas como metadado. |
| `size` | number | Tamanho do arquivo em bytes. Deve ser maior que zero e não exceder o limite configurado. |
| `uploadedAt` | string | Data/hora do upload em ISO 8601 UTC, por exemplo `2026-09-29T12:00:00.000Z`. |
| `owner` | string | Identificador obtido do header `X-User-Id`. |

O payload público não inclui caminho de filesystem, nome interno do arquivo, dados binários nem informação de armazenamento.

### Dados internos de persistência

O repositório em memória mantém os campos públicos e um campo interno `storageName`, um identificador opaco gerado pelo servidor para localizar o arquivo em `backend/storage`. `storageName` não pode ser serializado em respostas HTTP. O nome físico não deve incluir diretórios fornecidos pelo usuário nem depender de `originalName`.

O diretório pode conter arquivos após reinicializações, mas o processo não reconstrói metadados a partir deles. Limpeza automática de arquivos órfãos não faz parte do MVP.

## 6. Contratos de API

### Convenções comuns

- As rotas do backend são montadas sem prefixo `/api`. No desenvolvimento, o frontend chama `/api/...`; o proxy existente do Vite remove `/api` e encaminha a requisição para o backend.
- Todas as rotas de documentos exigem `X-User-Id`, após remoção de espaços nas extremidades. O valor deve conter de 1 a 128 caracteres e não conter caracteres de controle. A ausência ou invalidade resulta em `400 USER_ID_REQUIRED` ou `400 USER_ID_INVALID`.
- O header é uma identidade declarada pelo cliente, não uma credencial. Não usar o valor para alegar autenticação real.
- Erros usam o formato `{ "error": { "code": "...", "message": "..." } }`. Mensagens são estáveis, curtas e não incluem stack traces, caminhos locais ou dados de outros usuários.
- Não há paginação nesta versão. Listas são ordenadas por `uploadedAt` decrescente; empates podem ser desempatados por `id` em ordem crescente.

### `POST /upload`

**Entrada**

- Header obrigatório: `X-User-Id`.
- `Content-Type: multipart/form-data`.
- Um arquivo no campo `file`. Não aceitar múltiplos arquivos.
- Qualquer tipo de arquivo é aceito nesta fase. A aplicação não deve confiar no MIME declarado pelo cliente como uma validação de segurança.

**Sucesso: `201 Created`**

```json
{
  "document": {
    "id": "UUID-v4",
    "originalName": "relatorio.pdf",
    "size": 12345,
    "uploadedAt": "2026-09-29T12:00:00.000Z",
    "owner": "usuario-123"
  }
}
```

`UUID-v4` é ilustrativo; a resposta contém o UUID v4 real gerado pelo servidor. Não retornar `storageName` ou caminho físico.

**Erros**

| HTTP | Código | Quando |
| --- | --- | --- |
| `400` | `USER_ID_REQUIRED` / `USER_ID_INVALID` | Header obrigatório ausente ou inválido. |
| `400` | `FILE_REQUIRED` | Nenhum arquivo foi enviado. |
| `400` | `EMPTY_FILE` | O arquivo enviado tem zero bytes. |
| `400` | `INVALID_MULTIPART` | Multipart malformado, campo de arquivo incorreto ou mais de um arquivo. |
| `413` | `FILE_TOO_LARGE` | O arquivo excede `MAX_UPLOAD_SIZE_BYTES`. |
| `500` | `INTERNAL_ERROR` | Falha inesperada ao armazenar o arquivo ou registrar seus metadados. |

Em falha após a gravação física, o backend deve tentar remover o arquivo parcial/órfão. Se a limpeza também falhar, registrar a falha sem expor o caminho ao cliente.

### `GET /documents`

**Entrada**

- Header obrigatório: `X-User-Id`.
- Sem corpo ou parâmetros de consulta definidos nesta versão.

**Sucesso: `200 OK`**

```json
{
  "documents": [
    {
      "id": "UUID-v4",
      "originalName": "relatorio.pdf",
      "size": 12345,
      "uploadedAt": "2026-09-29T12:00:00.000Z",
      "owner": "usuario-123"
    }
  ]
}
```

Uma conta sem documentos recebe `200` com `"documents": []`. A lista inclui apenas documentos do owner informado.

**Erros**

| HTTP | Código | Quando |
| --- | --- | --- |
| `400` | `USER_ID_REQUIRED` / `USER_ID_INVALID` | Header obrigatório ausente ou inválido. |
| `500` | `INTERNAL_ERROR` | Falha inesperada ao consultar o repositório. |

### `GET /documents/:id/download`

**Entrada**

- Header obrigatório: `X-User-Id`.
- `id` é o UUID v4 do documento.

**Sucesso: `200 OK`**

- Corpo: bytes originais do arquivo.
- `Content-Type: application/octet-stream` nesta fase, sem confiar no MIME fornecido pelo cliente.
- `Content-Disposition: attachment` com `originalName` tratado/escapado como nome de download, nunca como caminho.
- Enviar `X-Content-Type-Options: nosniff`.

**Erros**

| HTTP | Código | Quando |
| --- | --- | --- |
| `400` | `USER_ID_REQUIRED` / `USER_ID_INVALID` | Header obrigatório ausente ou inválido. |
| `404` | `DOCUMENT_NOT_FOUND` | ID malformado, documento inexistente, owner diferente ou arquivo físico ausente. Usar a mesma resposta em todos esses casos. |
| `500` | `INTERNAL_ERROR` | Falha inesperada ao ler/abrir o arquivo antes de iniciar a resposta. |

Se ocorrer erro de leitura depois do início do streaming, encerrar a transmissão; não tentar escrever JSON em uma resposta binária já iniciada.

### Formato geral de erro

```json
{
  "error": {
    "code": "DOCUMENT_NOT_FOUND",
    "message": "Documento não encontrado."
  }
}
```

Os códigos apresentados nas tabelas são o conjunto mínimo para o MVP. Erros internos podem ser registrados no servidor, mas a resposta `500` não deve incluir detalhes internos.

## 7. Decisões arquiteturais

- **Backend:** Express em CommonJS, organizado em `routes/`, `controllers/`, `services/` e `repositories/`, no fluxo `routes → controllers → services → repositories`.
- **Routes:** declaram os endpoints e conectam o middleware de upload configurado com Multer. O middleware aceita um único arquivo no campo `file` e aplica o limite de tamanho.
- **Controllers:** validam e traduzem a entrada HTTP, chamam os serviços e definem status, headers e payloads HTTP. Não implementam regras de ownership nem persistência.
- **Services:** aplicam as regras de negócio, incluindo associação ao owner, visibilidade dos documentos e coordenação do upload/listagem/download. Não dependem de Express ou dos tipos do Multer.
- **Repositories:** encapsulam a coleção de metadados em memória e o acesso aos arquivos locais. Fornecem ao serviço somente os dados necessários, mantendo `storageName` privado da API.
- **Armazenamento:** Multer usa `diskStorage` e grava em `backend/storage`. O nome físico é gerado pelo servidor. Nenhum provedor externo é permitido.
- **Composição:** `app.js` monta rotas e configurações, preserva `/health` e lê `PORT` e `MAX_UPLOAD_SIZE_BYTES`. A configuração de `diskStorage` mantém o destino local fixo em `backend/storage`.
- **Frontend:** componentes React consomem os contratos por `fetch` e pelo prefixo `/api`, usando o proxy de desenvolvimento Vite já existente. O frontend exibe `originalName` como texto, sem interpretar HTML.
- **Erros:** middleware no limite HTTP converte erros de Multer e falhas conhecidas no formato comum da API; camadas internas não formatam respostas Express.

## 8. Plano de execução

As etapas abaixo são um roteiro para uma implementação futura. Esta entrega consiste somente nesta especificação; nenhuma etapa de código de backend ou frontend é executada agora.

1. **Preparar o backend:** configurar limite de upload por ambiente, garantir a existência do diretório local `backend/storage` e preparar a composição do Multer com `diskStorage` e nome físico gerado pelo servidor.
2. **Implementar persistência e regras:** criar repositório em memória para metadados e acesso local a arquivos; implementar serviços para upload, listagem filtrada por owner e download autorizado.
3. **Expor a API:** adicionar rotas e controllers dos três endpoints, validar identidade e entradas, mapear erros para os contratos desta spec e preservar `/health`.
4. **Validar o backend:** adicionar testes com `node:test` para upload válido e inválido, limite de tamanho, metadados, isolamento por owner, documento/arquivo inexistente, download binário e erros; garantir limpeza em falhas de upload.
5. **Implementar o frontend:** criar fluxo de upload, apresentação da lista do usuário e ação de download, chamando a API pelo prefixo `/api` e exibindo estados de carregamento e erros.
6. **Integrar e verificar:** executar testes do backend e build do frontend, validar o proxy Vite e percorrer upload → listagem → download manualmente, inclusive com dois valores distintos de `X-User-Id`.

## 9. Premissas e limites

- O owner vem do header `X-User-Id` para permitir testar isolamento no protótipo; qualquer cliente pode falsificá-lo. Uma solução real exige autenticação e autorização confiáveis antes de exposição a usuários.
- Os metadados em memória são a fonte de verdade da associação entre documento, owner e nome físico. Reiniciar o processo invalida essa associação, ainda que os bytes permaneçam no disco.
- Arquivos são limitados a 10 MiB por padrão e qualquer tipo é aceito. A ausência de allowlist não significa que o conteúdo seja seguro para abertura ou execução.
- Nomes e MIME fornecidos pelo cliente são dados não confiáveis. A aplicação não os usa para compor caminhos nem serve o arquivo inline.
- Sem banco, autenticação real, armazenamento externo, versionamento ou exclusão, conforme o escopo do MVP.