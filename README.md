## **Visão Geral**
### **queue-app-cs-plugin**

O **queue-app-cs-plugin** adiciona em uma stack a capacidade de provisionar o uso da Amazon Simple Queue Service (SQS) seja consumindo ou produzindo mensagens na fila.

![Caso de Uso](caso-uso-1.png "Caso de Uso")

Para aplicar o plugin execute o comando:
```
stk apply plugin skynet-dotnet-stack/queue-app-cs-plugin
```

#### **Pré-requisitos**
Para utilizar esse plugin é necessário ter uma stack dotnet criada pelo cli do StackSpot que você pode baixar [**aqui**](https://stackspot.com.br/).

Ter instalado:
- .NET 5 ou 6 
- O template base de `rest-app-cs-template` já deverá estar aplicado para você conseguir utilizar este plugin. 

#### **Inputs**

* RegionEndpoint - Endpoint regional que será utilizado para requisitar o SQS - Campo Obrigatório.

Você pode sobrescrever a configuração padrão adicionando a seção `Sqs` em seu `appsettings.json`. Os valores aceitáveis você pode encontrar [aqui](https://docs.aws.amazon.com/pt_br/pt_br/AWSEC2/latest/WindowsGuide/using-regions-availability-zones.html#concepts-available-regions).

```json
  "Sqs": {
      "RegionEndpoint": "us-east-1"
  }
```

Adicione ao seu `IServiceCollection` via `services.AddSQS()` no `Startup` da aplicação ou `Program` tendo como parametro de entrada `IConfiguration` e `IWebHostEnvironment`. 

```csharp
services.AddSQS(Configuration, Env);
```

A  classe da mensagem que será integrada na fila, deverá herdar da classe `MessageBase` e decorada com `Queue`, especificando o nome e o tipo da fila, que pode ser Standard ou Fifo.
- Enqueue - Publica uma mensagem na fila. Filas do tipo Fifo, possuem um identificador único.
- Dequeue - Remove uma mensagem da fila.
- Receive - Retorna uma mensagem da fila.
- ReceiveMessages - Retorna uma lista de mensagens da fila.

Metodo que retorna uma mensagem utilizando o Receive.

```csharp
    [Queue("Test", QueueType.Standard)]
    public class Test : MessageBase
    {
        public Test(string correlationId) : base(correlationId) { }
    }

    [ApiController]
    [Route("[controller]")]
    public class SampleController : ControllerBase
    {
        private readonly IQueueSqs _queueSqs;

        public SampleController(IQueueSqs _queueSqs)
        {
            _queueSqs = queueSqs;
        }

        [HttpGet]
        public async Task<IActionResult> Get()
        {
            var test = new Test(Guid.NewGuid().ToString());
            _queueSqs.Enqueue(test);

            var message = _queueSqs.Receive<Test>();

            _queueSqs.Dequeue(test);

            return Ok(message);
        }
    }
```

Metodo que retorna uma lista de mensagens utilizando o ReceiveMessages.

Paramentros do MessageRequest

`MaxNumberOfMessages` O número máximo de mensagens a serem retornadas. O Amazon SQS nunca retorna mais mensagens do que
este valor (no entanto, menos mensagens podem ser retornadas). Valores válidos: 1 a 10. Padrão:1.

`WaitTimeSeconds` A duração (em segundos) pela qual a chamada aguarda a chegada de uma mensagem no
fila antes de retornar.

`VisibilityTimeout` A duração (em segundos) em que as mensagens recebidas ficam ocultas da recuperação subsequente
solicitações após serem recuperadas por um ReceiveMessage.

`ReceiveRequestAttemptId` O ID de tentativa de solicitação de recebimento é o token usado para deduplicação de chamadas ReceiveMessage.

```csharp
    [Queue("Test", QueueType.Standard)]
    public class Test : MessageBase
    {
        public Test(string correlationId) : base(correlationId) { }
    }

    [ApiController]
    [Route("[controller]")]
    public class SampleController : ControllerBase
    {
        private readonly IQueueSqs _queueSqs;

        public SampleController(IQueueSqs _queueSqs)
        {
            _queueSqs = queueSqs;
        }

        [HttpGet]
        public async Task<IActionResult> GetListMessage()
        {
            var messageRequest = new MessageRequest
            {
                MaxNumberOfMessages = 2,
                VisibilityTimeout = 10,
                WaitTimeSeconds = 1
            };

            var messages = await _queueSqs.ReceiveMessages<Test>(messageRequest);

            return Ok(messages);
        }
    }
```

#### Ambiente local

- Esta etapa não é obrigatória.
- Recomendamos, para o desenvolvimento local, a criação de um contâiner com a imagem do [Localstack](https://github.com/localstack/localstack). 
- Para o funcionamento local você deve preencher a variável de ambiente `LOCALSTACK_CUSTOM_SERVICE_URL` com o valor da url do serviço. O valor padrão do localstack é http://localhost:4566.
- Abaixo um exemplo de arquivo `docker-compose` com a criação do contâiner: 

```yaml
    version: '2.1'

    services:
    localstack:
        image: localstack/localstack
        ports:
        - "4566:4566"
        environment:
        - SERVICES=sqs
        - AWS_DEFAULT_OUTPUT=json
        - DEFAULT_REGION=us-east-1
```

Após a criação do contâiner, crie uma fila para realizar os testes com o componente. Recomendamos que você tenha instalado em sua estação o [AWS CLI](https://aws.amazon.com/pt/cli/). Abaixo um exemplo de comando para criação de uma fila:

```bash
    aws --endpoint-url=http://localhost:4566 --region=us-east-1 sqs create-queue --queue-name [NOME DA SUA FILA]
```
