# Midware: Documentação de Produto (Trabalho de Conclusão de Curso)

## 1. Visão geral

### O que é

Midware é uma plataforma backend de mensageria: um middleware que centraliza o envio de mensagens transacionais a usuários finais, sem interface própria voltada ao usuário final. Qualquer aplicação cliente aciona essa plataforma via chamada de API, ao invés de implementar e manter sua própria integração direta com os provedores de WhatsApp e e-mail.

### Propósito

Resolver, uma única vez e de forma reutilizável, um problema que se repete em qualquer sistema de negócio que precisa notificar seus usuários: como enviar uma mensagem, por qual canal, com qual template, e como saber depois se ela chegou.

### Canais suportados

1. WhatsApp, via Meta Cloud API.
2. E-mail transacional.

### Características gerais

1. Multi-tenant desde a concepção: cada tenant se autentica através de uma chave (key) enviada na própria chamada, por exemplo `{{url}}/transactions?key={{key}}`. É essa chave que identifica o tenant e garante o isolamento de dados no registro de envios. Autenticar via query string é um débito técnico conhecido, ver seção 10.
2. O catálogo de tipos de mensagem é fechado, limitado a um pequeno conjunto de mensagens pré-cadastradas pelo Midware (cinco, ver seção 8). O tenant não cadastra mensagens próprias, apenas escolhe entre as existentes através do campo `messageId`.
3. O tenant pode consultar o catálogo de mensagens disponíveis, e o `messageId` de cada uma, através de uma API própria de consulta.
4. Cada chamada de envio carrega dois campos de identificação: `channelId`, que indica o canal da mensagem (1 para WhatsApp, 2 para e-mail, 3 para os dois), e `messageId`, que indica qual das mensagens do catálogo deve ser usada. Os dois campos são independentes: o mesmo `messageId` pode ser enviado por qualquer `channelId`.
5. O corpo (body) de cada chamada também carrega os dados customizados da mensagem: o destinatário e as variáveis que o template daquele `messageId` vai usar, como número do pedido e link.
6. Quando `channelId` é 1 ou 2 (um canal só), o tenant pode opcionalmente marcar `fallback: true`, pedindo que o Midware tente o canal alternativo em caso de falha no canal principal, desde que o dado do canal alternativo também tenha sido enviado. Quando `channelId` é 3, o campo `fallback` é ignorado, porque os dois canais já são tentados de qualquer forma.
7. Para envios em lote, o corpo carrega uma lista de destinatários (`recipients`), cada um com seus próprios dados de contato e variáveis, sob o mesmo `channelId` e `messageId` do lote inteiro.
8. Para o canal WhatsApp, todos os tenants enviam usando um único número/linha corporativo do Midware, não uma linha própria por tenant. Um único WhatsApp Business Account, com templates aprovados uma vez, serve para todos.
9. Para o canal e-mail vale a mesma lógica: todos os tenants enviam a partir de um único remetente do Midware, configurado no próprio serviço, sem que o tenant informe remetente na chamada. Neste TCC é usado um remetente de teste. Em um produto comercial seria necessário um domínio corporativo próprio, verificado junto ao provedor de e-mail.
10. Serviço deliberadamente "burro" quanto ao destinatário e ao momento do envio: quem aciona o Midware sempre resolve para quem enviar e decide quando enviar. O Midware só sabe qual template usar e como preenchê-lo.

---

## 2. Diagrama de arquitetura

![Middleware Diagram](src/image/MiddlewareDiagram.png)


---

## 3. Conceitos-chave

Antes de descrever os fluxos, três conceitos precisam estar claros, porque são referenciados o tempo todo no restante do documento.

1. Tenant: a aplicação cliente que está integrada ao Midware, identificada pela key enviada em cada chamada.
2. Transação: uma mensagem individual, seja ela enviada sozinha (envio isolado) ou como parte de um lote (envio em lote). Toda transação tem um `transactionId` próprio, gerado pelo Midware no momento do processamento, nunca informado pelo tenant.
3. Lote: um agrupamento de transações enviadas juntas, de uma vez. Todo lote tem um `batchId` próprio, também gerado pelo Midware.

A relação entre os dois: toda transação tem um `transactionId`. Quando a transação veio de um lote, ela também carrega o `batchId` ao qual pertence. Quando a transação veio de um envio isolado, o campo `batchId` fica nulo. Isso permite responder, para qualquer transação, se ela veio de uma chamada isolada ou de um lote, e qual lote.

Como o tenant não sabe o `transactionId` no momento do envio, já que ele só existe depois que o Midware processa a chamada, a resposta do envio em lote devolve, para cada destinatário enviado, o `transactionId` gerado e o `index` dele na lista original, usado para o tenant relacionar a resposta com o destinatário que ele mandou.

Exemplo ilustrativo, resposta de um lote com dois destinatários:

```json
{
  "batchId": "b1e2c3d4-8f3a-4a2e-9c1d-7e6f5a4b3c2d",
  "transactions": [
    { "transactionId": "a1b2c3d4-...", "index": 0, "status": "queued" },
    { "transactionId": "f47ac10b-58cc-4372-a567-0e02b2c3d479", "index": 1, "status": "queued" }
  ]
}
```

Com isso, a consulta de status por `transactionId` sempre traz, junto com os dados da transação, o `batchId` correspondente quando ela pertence a um lote, ou `null` quando é um envio isolado.

---

## 4. Princípios de arquitetura

1. Tenant resolvido pela key enviada em cada chamada, nunca fixo no código.
2. Toda tentativa de envio é registrada em banco (não em memória), com o ID do provedor (quando houver), o `transactionId`, o `batchId` (quando houver), o ID do tenant e o status do envio. Registrar também as falhas é essencial, por exemplo para identificar um usuário que nunca recebeu uma notificação por causa de um número de telefone cadastrado errado.
3. Abstração de tipo de mensagem: cada `messageId` do catálogo mapeia para um template aprovado, permitindo trocar o conteúdo do template no futuro sem impactar quem consome o serviço. O canal (`channelId`) é independente do template, escolhido pelo tenant a cada chamada.
4. Princípios gerais de arquitetura de software valem igualmente aqui: SQL parametrizado, injeção de dependência, separação de responsabilidades.

---

## 5. Stack tecnológica

1. .NET 10, C#.
2. MySQL como banco de dados.
3. Arquitetura em camadas (N-tier).
4. xUnit para testes unitários.
5. Docker para containerização.
6. Hospedagem no Railway.
7. Meta Cloud API para envio por WhatsApp.
8. Resend para envio por e-mail.

---

## 6. Envio pontual de mensagem (API síncrona)

### O que é

Canal de envio para mensagens individuais e imediatas, onde quem aciona precisa de confirmação de que o envio foi aceito para processamento.

### Como funciona

1. Aplicação cliente (ex: um sistema de gestão de frotas ou de agendamentos) chama a API do Midware informando o `messageId` da mensagem catalogada, o `channelId`, opcionalmente `fallback`, o `callbackUrl`, o destinatário e as variáveis a preencher.
2. Midware valida a chamada e mapeia o `messageId` para o template aprovado correspondente.
3. Midware gera um `transactionId` e registra a transação no banco com status `queued` (tenant, destinatário, `messageId`, `channelId`, `transactionId`, `batchId` nulo, `callbackUrl`, data/hora).
4. Midware retorna `202 Accepted` com o `transactionId` e o status `queued` para quem chamou.
5. Midware executa o envio via provedor (Meta Cloud API para WhatsApp, provedor de e-mail transacional para e-mail), conforme o `channelId` informado, e registra na transação o ID devolvido pelo provedor.
6. Quando o status da transação mudar (ex: enviado, entregue, falhou), o Midware notifica a aplicação cliente através de uma chamada de callback para a `callbackUrl` informada naquela transação.

### Regras de negócio importantes

1. Usado para mensagens que precisam ser entregues imediatamente e onde uma única pessoa é notificada por vez, como o convite de primeiro acesso e o reset de senha.
2. O Midware não decide se deve enviar ou não, apenas executa e confirma. A decisão de acionar já foi tomada por quem chamou.
3. O `callbackUrl` é informado a cada chamada, não é um dado cadastrado previamente pelo tenant. Se vier ausente, o Midware apenas não notifica, o tenant fica dependendo da consulta manual de status.

### Contrato técnico

**Envio da mensagem:**

```json
POST /v1/transactions?key={{key}}

{
  "channelId": 3,
  "messageId": 5,
  "fallback": true,
  "callbackUrl": "https://tenant.example.com/webhooks/midware",
  "recipient": {
    "name": "Maria Silva",
    "phone": "+353123456789",
    "email": "maria@example.com",
    "variables": {
      "registrationNumber": "12345",
      "institutionName": "Instituto Exemplo",
      "link": "https://example.com/visa-renewal/12345"
    }
  }
}
```

```json
202 Accepted

{
  "transactionId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "status": "queued"
}
```

**Callback (notificação de mudança de status):**

```json
POST {callbackUrl}

{
  "transactionId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "batchId": null,
  "status": "delivered",
  "updatedAt": "2026-09-23T14:02:04Z"
}
```

**Consulta de status:**

```json
GET /v1/transactions/f47ac10b-58cc-4372-a567-0e02b2c3d479?key={{key}}

200 OK

{
  "transactionId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "batchId": null,
  "messageId": 5,
  "channelId": 3,
  "recipient": {
    "name": "Maria Silva",
    "phone": "+353123456789",
    "email": "maria@example.com"
  },
  "status": "delivered",
  "createdAt": "2026-09-23T14:02:00Z",
  "updatedAt": "2026-09-23T14:02:04Z"
}
```

### Padrão de integração

Chamada síncrona de API, consumida por aplicações clientes em fluxos como autenticação e recuperação de acesso.

### Pontos em aberto

1. Como proteger a aplicação contra ataques de negação de serviço (DDoS), ou contra má implementação de parceiro, onde, em vez de aguardar o callback, o tenant fica consultando o status da transação repetidamente em curto intervalo de tempo.

---

## 7. Envio em massa de mensagem (lote)

### O que é

Canal de envio para lembretes ou notificações disparadas para muitos destinatários de uma vez, sem necessidade de resposta imediata de cada transação individual.

### Como funciona

1. Aplicação cliente publica um lote informando o `messageId`, o `channelId`, opcionalmente `fallback` e `callbackUrl`, válidos para o lote inteiro, e a lista `recipients` com os dados e as variáveis de cada destinatário.
2. Midware gera um `batchId` para o conjunto inteiro, e um `transactionId` para cada destinatário dentro dele.
3. A aplicação cliente atua como produtora da fila, o Midware como consumidor.
4. Midware processa a fila de forma assíncrona, respeitando os limites de taxa do provedor.
5. Cada transação processada é registrada no banco com seu `transactionId` e o `batchId` ao qual pertence, do mesmo jeito que no envio isolado, mas com o vínculo ao lote preenchido.
6. Quando o status de cada transação do lote mudar, o Midware notifica a aplicação cliente através de uma chamada de callback para a `callbackUrl` informada, uma chamada por transação.

### Regras de negócio importantes

Usado para casos como lembretes periódicos, onde a aplicação cliente varre diariamente uma condição de negócio (ex: pendências, prazos vencidos) e dispara uma notificação para cada usuário afetado de uma vez.

Exemplo: a aplicação cliente controla quem tem visto válido, vencido ou a vencer. A aplicação cliente gera uma lista de usuários com visto vencido e manda um lote de notificações informando que a pessoa precisa regularizar a situação, e que até lá o cadastro ficará suspenso.

O `callbackUrl` é informado a cada chamada de lote, não é um dado cadastrado previamente pelo tenant. Se vier ausente, o Midware apenas não notifica, o tenant fica dependendo da consulta manual de status.

### Contrato técnico

**Envio do lote:**

```json
POST /v1/batches?key={{key}}

{
  "channelId": 1,
  "messageId": 5,
  "fallback": true,
  "callbackUrl": "https://tenant.example.com/webhooks/midware",
  "recipients": [
    {
      "name": "Maria Silva",
      "phone": "+353123456789",
      "email": "maria@example.com",
      "variables": {
        "registrationNumber": "12345",
        "institutionName": "Instituto Exemplo",
        "link": "https://example.com/visa-renewal/12345"
      }
    },
    {
      "name": "Eva Silva",
      "phone": "+353123456780",
      "email": "eva@example.com",
      "variables": {
        "registrationNumber": "54321",
        "institutionName": "Instituto Exemplo",
        "link": "https://example.com/visa-renewal/54321"
      }
    }
  ]
}
```

```json
202 Accepted

{
  "batchId": "b1e2c3d4-8f3a-4a2e-9c1d-7e6f5a4b3c2d",
  "transactions": [
    { "transactionId": "a1b2c3d4-...", "index": 0, "status": "queued" },
    { "transactionId": "f47ac10b-58cc-4372-a567-0e02b2c3d479", "index": 1, "status": "queued" }
  ]
}
```

**Callback (notificação de mudança de status, uma por transação):**

```json
POST {callbackUrl}

{
  "transactionId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "batchId": "b1e2c3d4-8f3a-4a2e-9c1d-7e6f5a4b3c2d",
  "status": "delivered",
  "updatedAt": "2026-09-23T14:02:04Z"
}
```

**Consulta de status de uma transação do lote:** segue o mesmo contrato já descrito na seção 6, `GET /v1/transactions/{transactionId}?key={{key}}`, com o `batchId` preenchido em vez de nulo.

### Padrão de integração

Fila de mensagens (mensageria assíncrona), com a aplicação cliente como produtora e o Midware como consumidor.

### Pontos em aberto

1. Escolha da tecnologia/broker de fila ainda não definida. Fica para quando a implementação técnica for iniciada, não é uma decisão de produto.
2. Se vai existir uma consulta agregada por `batchId` (status geral do lote inteiro: quantas transações enviadas, falharam, pendentes), ou se o tenant sempre consulta transação por transação. Ainda não decidimos.
3. As mesmas preocupações de proteção contra DDoS e consulta repetida em sequência, registradas na seção anterior, valem aqui também, possivelmente de forma ainda mais crítica, já que um único lote pode gerar centenas de transações de uma vez.

---

## 8. Tipos de mensagem (contrato)

### O que é

Cada tipo de mensagem suportado pelo Midware tem um `messageId` próprio no catálogo, mapeando para um template de conteúdo aprovado. O canal de envio (`channelId`) é escolhido pelo tenant a cada chamada, independente do `messageId`: o mesmo template pode ser enviado por WhatsApp, e-mail, ou os dois, desde que o conteúdo não dependa de particularidades de um canal específico.

### Catálogo de mensagens (fechado, cinco tipos)

| messageId | Nome | Descrição | Variáveis exigidas |
|---|---|---|---|
| 1 | First Access Invite | Enviado quando um novo usuário é cadastrado pela aplicação cliente, contendo o link para definição de senha. | `name`, `link` |
| 2 | Password Reset | Enviado quando um usuário solicita redefinir a senha, contendo o link com token. | `name`, `link` |
| 3 | Pending Reminder | Enviado aos usuários que ainda não completaram uma ação esperada dentro do prazo. | `name`, `link` |
| 4 | Order Shipped | Enviado quando um pedido do usuário sai para entrega. | `name`, `orderNumber`, `link` |
| 5 | Visa Expiration Notice | Enviado a usuários com visto de residência próximo do vencimento. | `name`, `registrationNumber`, `institutionName`, `link` |

Todos os cinco se enquadram na categoria Utility (utilidade) do WhatsApp Business, categoria de aprovação mais simples por não ter caráter promocional, mantendo o mesmo tom direto e informativo do template já em produção na conta.

### Templates aprovados (mensagens ilustrativas)

**messageId 1, First Access Invite:**
```
Hi {{name}}, your access to the system has been created. To set your
password and get started, please use the link below: {{link}}
```

**messageId 2, Password Reset:**
```
Hi {{name}}, we received a request to reset your password. To
continue, use the link below: {{link}}. If you didn't request this,
you can safely ignore this message.
```

**messageId 3, Pending Reminder:**
```
Hi {{name}}, we haven't received your update yet for today. Please
take a moment to complete your pending record so everything stays
up to date. You can access the system here: {{link}}
```

**messageId 4, Order Shipped:**
```
Hi {{name}}, your order {{orderNumber}} has been shipped. For more
details, you can track it here: {{link}}
```

**messageId 5, Visa Expiration Notice:**
```
Hi {{name}}, we noticed that your residence visa is about to expire.
Your registration number {{registrationNumber}} is linked to this
document. If you have already started your renewal process, please
access the link below and complete the requested steps: {{link}}.
Regards, {{institutionName}}
```

### Contrato técnico

**Consulta do catálogo de mensagens:**

```json
GET /v1/message-types?key={{key}}

200 OK

[
  { "messageId": 1, "name": "First Access Invite" },
  { "messageId": 2, "name": "Password Reset" },
  { "messageId": 3, "name": "Pending Reminder" },
  { "messageId": 4, "name": "Order Shipped" },
  { "messageId": 5, "name": "Visa Expiration Notice" }
]
```

### Regras de negócio importantes

1. O catálogo é fechado, limitado a essas cinco mensagens. Novos tipos exigiriam alteração de escopo do Midware, não é algo que o tenant cadastra por conta própria.
2. Em todos os tipos de mensagem, é sempre quem aciona o Midware que resolve o destinatário (número de telefone ou endereço de e-mail) antes da chamada. O Midware nunca decide ou descobre por conta própria para quem enviar.
3. O campo `name` do destinatário preenche automaticamente a variável `{{name}}` do template, sem precisar ser repetido dentro de `variables`. As demais variáveis exigidas pelo template (ver tabela do catálogo) vêm do objeto `variables` enviado na chamada.
4. Em todos os tipos de mensagem, o link vem inteiramente como variável fornecida pelo tenant na chamada. O Midware apenas insere o valor no template, sem gerar ou validar o conteúdo do link. No caso do convite de primeiro acesso e do reset de senha, a geração e a validação do token de uso único também são responsabilidade do tenant..

### Pontos em aberto

O WhatsApp Business API exige que o texto do template esteja pré-aprovado pela Meta, com pouca flexibilidade de formatação. O e-mail não tem essa restrição. Como o mesmo `messageId` pode ser enviado por qualquer um dos dois canais, fica em aberto se cada mensagem terá uma versão de template por canal (uma para WhatsApp, outra para e-mail), ou se o texto será único e restrito ao formato mais limitado dos dois, mesmo quando enviado por e-mail.

---

## 9. Lista de APIs expostas

Consolidação de todos os endpoints definidos nas seções anteriores. Todas as chamadas exigem o parâmetro `key` na query string (`?key={{key}}`), conforme descrito na seção 1.

| Método | Endpoint | Descrição | Detalhado na seção |
|---|---|---|---|
| POST | `/v1/transactions` | Envio pontual de uma mensagem | 6 |
| GET | `/v1/transactions/{transactionId}` | Consulta de status de uma transação (isolada ou de lote) | 6 |
| POST | `/v1/batches` | Envio de um lote de mensagens | 7 |
| GET | `/v1/message-types` | Consulta do catálogo de mensagens disponíveis | 8 |

Além disso, o Midware é quem faz uma chamada de saída, não exposta como API própria: o callback (`POST {callbackUrl}`), disparado para a URL informada pelo tenant a cada transação, sempre que o status dela muda.

---

## 10. Pendências técnicas

1. Escolha da tecnologia/broker de fila para envios em massa (ver também seção 7).
2. Proteção contra DDoS e contra consulta de status em sequência abusiva (ver também seções 6 e 7).
3. Autenticação via query string (`?key={{key}}`) é a melhor forma? como empresas realizam esse processo?
5. Tempo limite de transações sem retorno do provedor: definir após quanto tempo uma transação pendente é marcada como falha. Esse tempo também define quando o fallback é acionado.
6. Segurança dos webhooks recebidos: validar a assinatura enviada pela Meta e pelo Resend, para que só eles consigam atualizar o status de uma transação.
7. Segurança dos callbacks enviados: assinar os callbacks do Midware, para que o tenant consiga confirmar que o aviso veio realmente do Midware. 
8. Status possíveis de uma transação: definir a lista oficial e o significado de cada um, por exemplo `queued`, `sent`, `delivered` e `failed`, deixando claro que `delivered` significa entregue ao destino, sem garantia de caixa de entrada ou leitura. 
9. Modelagem do banco de dados, definir as tabelas, campos e relacionamentos (tenants, transações, lotes, catálogo de mensagens), a partir dos dados já descritos nas seções 3, 4 e 6.

---