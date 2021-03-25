+++
date = "2020-06-10"
title = "Implementando OpenTracing com Jaeger em microservices utilizando C# & AWS SQS [pt-BR]"
+++

**Artigo publicado pela primeira vez no [InfoQ](https://www.infoq.com/br/articles/implementando-opentracing-jager-dotnet-sqs/)**

`Pontos Principais`

- Lidar com dados e transações faz com que desenvolvedores adotem padrões orientados a eventos/comandos. A utilização de serviços de filas e tópicos é muitas vezes uma boa solução, mas acaba elevando a complexidade de rastreamento das nossas aplicações.
- A complexidade da execução de um tracing depende do modelo de concorrência implementado.
- Em muitos cenários de troubleshooting em sistemas distribuídos, apenas os loggings da aplicação não são suficientes em modelos de concorrência distribuída

#### Sempre surgirão novos desafios

O desenvolvimento de serviços distribuídos, a ocorrência de falhas independentes e o não determinismo causam os problemas importantes em sistemas distribuídos. Além das falhas de computação típicas, com as quais a maioria de times de engenharia está acostumada a lidar, as falhas podem ocorrer de várias outras maneiras. Entender o comportamento dos serviços entre um request/job se torna algo desafiador.

Sabemos que os desafios que desenvolvedores enfrentam são maiores em cenários onde o mecanismo para garantir a consistência das transações entre os serviços que se comunicam através de eventos como o padrão Saga se faz necessário.

Este artigo discorre sobre a utilização do OpenTracing em contextos como os descritos até aqui, bem como ilustra a implementação dessa solução em C# .NET Core juntamente com o SQS da Amazon.

#### Concorrência distribuída

A vantagem de dispor de uma maior capacidade de processamento e, consequentemente, suportar um número maior de usuários/requisições/processos é uma das vantagens em um modelo de concorrência distribuída.

![Istio](/images/concorrencia-distribuida.jpg)

No entanto, como toda solução em computação tem seus prós e contras, o modelo de concorrência distribuída trás junto de si algumas dificuldades, tais como:

1. Comportamento pouco previsível, dependência de computadores interligados em rede. Sabemos que a rede não é homogênea e comportamentos inesperados, como latência e quedas repentinas de conexão, podem surgir a qualquer momento.
2. Em geral os relógios dos servidores não estão sincronizados, além de não serem precisos. Sempre há um desvio entre o tempo real e o marcado pelo relógio. Relógios em máquinas diferentes tendem a perder a sincronia com o passar do tempo, mesmo que sincronizados em um dado momento.
3. É difícil ter uma visão global dos microservices, dado a velocidade em que são desenvolvidos novos serviços. A gestão do software tende a aumentar consideravelmente sua complexidade em virtude da quantidade excessiva de microservices.

#### O que é OpenTracing?

![Opentracing](/images/opentracing.png)

*[Ecossistema OpenTracing - [Depth, Breadth, and the Future of Tracing - Ben Sigelman](https://www.youtube.com/watch?v=n8mUiLIXkto)]*

Como menciona o site oficial https://opentracing.io/, OpenTracing não é um padrão, nem um programa para ser instalado.

> "O OpenTracing é composto por uma especificação de API, estruturas e bibliotecas que implementam a especificação e documentação para o projeto. O OpenTracing permite que os desenvolvedores adicionem instrumentação ao código do aplicativo usando APIs que não os bloqueiam em nenhum produto ou fornecedor em particular."

Em teoria, uma premissa fundamental de uma arquitetura de microservices é que a medida que os requisitos de negócios aumentam e se tornam mais complexos, a estrutura da organização também cresce. A decomposição de funcionalidades de negócios em serviços independentes permite que pequenos times atuem com autonomia, cada um com o próprio ciclo de vida.

Portanto, a maior granularidade dos microservices torna a visibilidade de como cada time / serviço interage com os outros um tanto quanto obscura e, consequentemente, um desafio para os times em ter o mínimo de observabilidade. Nesses casos, distributed tracing torna-se uma ferramenta indispensável para depurar.

Com uma infraestrutura bem planejada é possível recuperar todas as interações entre os serviços, possibilitando um troubleshooting com detalhes onde apenas os loggings das aplicações não seriam suficientes. Isso não quer dizer que os loggings das aplicações são dispensáveis. Pelo contrário, eles se tornam ferramentas complementares.

#### OpenTracing / Amazon SQS

##### MOM - Sistemas de Mensageira

O [AWS SQS](https://aws.amazon.com/pt/sqs/) é um serviço gerenciado de filas de mensagens que permite enviar e receber mensagens entre componentes de software e que permite a criação de dois tipos de filas: padrão e FIFO. A diferença entre eles é que filas do tipo FIFO permitem que as mensagens sejam processadas apenas uma vez, na ordem exata em que foram enviadas.

##### Evitando a síndrome NIH (Not-invented-here)

Quando nos deparamos com novos desafios de implementação, podemos buscar na comunidade de desenvolvimento de software se já houve menção ao problema em questão, evitando assim invenções imperfeitas ou NIH. No site oficial existe [uma busca de contribuições](https://opentracing.io/registry/) para produtos específicos, onde podemos encontrar se a implementação necessária já foi realizada. Para a instrumentação utilizando AWS SQS existe uma biblioteca escrita em Java, mas não existe uma implementação em C# .NET Core.

#### Jaeger: open source
![Jaeger](/images/jaeger.png)

*[Arquitetura Jaeger - [https://www.jaegertracing.io/docs/1.17/architecture/](https://www.jaegertracing.io/docs/1.17/architecture)]*

A arquitetura do Jaeger é definida em cima do modelo de dados do OpenTracing. A leitura da [especificação](https://github.com/opentracing/specification/blob/master/specification.md) ajudará a entender melhor seu funcionamento. Em linhas gerais, os Traces são definidos implicitamente por seus Spans.

![Jaeger](/images/span.png)

*[Relação entre Spans e um único Trace - A especificação semântica do OpenTracing]*

Os três principais componentes envolvidos em uma implementação OpenTracing são:

- **Trace**: Descrição de uma transação à medida que ela se move através dos serviços.
- **Span**: É uma operação que representa uma parte do fluxo. Os Spans aceitam tags chave:valores, bem como logs estruturados com registro de data e hora, anexados a uma instância de span específica.
- **Span Context**: Contém as informações que acompanham a transação, inclusive quando ela passa de serviço em serviço pela rede ou por um serviço de enfileiramento de mensagens.

#### C# .NET Core - Implementação

Após entender esses três componentes, o próximo passo é compreender o método **Inject**, que permite que o mesmo **SpanContext** seja repassado para a próxima requisição.

```csharp
public async Task<SendMessageResponse> Enqueue(IntegrationEvent @event)
{
    var operationName = "SQS::SendMessageAsync/";
    var eventName = $"{operationName}{@event.GetType().Name.ReplaceSufixEvent().ToLower()}";

    using (var scope = tracer.BuildSpan(eventName).StartActive(true))
    {
        var span = scope.Span.SetTag(Tags.SpanKind, Tags.SpanKindProducer);
        var attributes = new Dictionary<string, string>();

        tracer.Inject(span.Context, BuiltinFormats.TextMap, new TextMapInjectAdapter(attributes));
        @event.MessageAttributes = attributes;

        logger.LogInformation($"Enqueue::{DateTime.UtcNow}|{eventName} SpanId:{scope.Span.Context.SpanId} TraceId:{scope.Span.Context.TraceId}");

        return await eventBus.Enqueue(@event);
    }
}
```

No entanto, como estamos utilizando AWS SQS, precisamos fazer com que os identificadores da transação sejam gravados na mensagem que irá para o serviço de enfileiramento de mensagens.

O AWS SQS possui uma feature onde podemos passar atributos customizados dentro da mensagem, chamado **MessageAttributeValue**. Maiores informações sobre essa funcionalidade e sua especificação podem ser encontradas na [documentação oficial](https://docs.aws.amazon.com/pt_br/AWSSimpleQueueService/latest/APIReference/API_MessageAttributeValue.html).

Com um método de extensão, convertemos um Dictionary com os identificadores da transação em um modelo de atributos do AWS SQS.

```csharp
internal static void BuildToMessageAttribute(this Dictionary<string, string> keyValues, Dictionary<string, MessageAttributeValue> messagesAttribute)
{
    if (keyValues != null)
    {
        foreach (KeyValuePair<string, string> entry in keyValues)
        {
            messagesAttribute.Add(entry.Key, new MessageAttributeValue()
            {
                DataType = "String",
                StringValue = entry.Value
            });
        }
    }
}
```

Ponto importante: após recuperar uma mensagem do AWS SQS, recuperamos os identificadores que foram postados via MessageAttributeValue e, utilizando método Extract, extraímos o SpanContext do cliente, utilizando esse objeto para ativar a thread principal via ScopeManager.

```csharp
public TEvent ReceiveMessage<TEvent>(int waitTimeSeconds) where TEvent : IntegrationEvent
{
    var message = eventBus.ReceiveMessage<TEvent>(waitTimeSeconds);
    if (message != null)
    {
        var operationName = "SQS::ReceiveMessageAsync/";
        var eventName = $"{operationName}{typeof(TEvent).Name.ReplaceSufixEvent().ToLower()}";

        using (var scope = tracer.StartSpanConsumer(message.MessageAttributes, eventName))
        {
            tracer.ScopeManager.Activate(tracer.ActiveSpan, false);
            logger.LogInformation($"ReceiveMessage::{DateTime.UtcNow} | {eventName} SpanId:{scope.Span.Context.SpanId} TraceId:{scope.Span.Context.TraceId}");

            return message;
        }
    }
    return message;
}
```

Agora se faz necessário implementar o método de extensão que extrai o SpanContext do lado do cliente.

```csharp
internal static IScope StartSpanConsumer(this ITracer tracer, IDictionary>string, string=""< messageAttributes, string operationName)
{
    ISpanBuilder spanBuilder;
    try
    {
        ISpanContext spanContext = tracer.Extract(BuiltinFormats.TextMap, new TextMapExtractAdapter(messageAttributes));

        spanBuilder = tracer.BuildSpan(operationName);
        if (spanContext != null)
        {
            spanBuilder = spanBuilder.AsChildOf(spanContext);
        }
    }
    catch (Exception)
    {
        spanBuilder = tracer.BuildSpan(operationName);
    }
    return spanBuilder.WithTag(Tags.SpanKind, Tags.SpanKindConsumer).StartActive(true);
}
```

O importante é que em todos os métodos implementados até aqui estamos criando logging do TraceId e SpanId. Isso permite que possamos correlacionar os logs da aplicação com o distributed tracing.

```csharp
logger.LogInformation($"ReceiveMessage::{DateTime.UtcNow} | {eventName} SpanId:{scope.Span.Context.SpanId} TraceId:{scope.Span.Context.TraceId}");
```

A utilização e instalação é bastante simples, ela pode ser feita via NuGet package

```bash
dotnet add package EventBus.Sqs.Tracing
```

Após instalar, é necessário registrar o tracer na nossa aplicação:

```csharp
using OpenTracing.Util;
...
services.AddSingleton(serviceProvider =>
{
    var loggerFactory = new LoggerFactory();

    var config = Jaeger.Configuration.FromEnv(loggerFactory);
    var tracer = config.GetTracer();

    if (!GlobalTracer.IsRegistered())
        GlobalTracer.Register(tracer);
    return tracer;
});
```

E, por fim, adicionar a configuração para utilização do AWS SQS com *distributed tracing*.

```csharp
using EventBus.Sqs.Configuration;
using EventBus.Sqs.Tracing.Configuration;
...
services.AddEventBusSQS(Configuration).AddOpenTracing();

[ApiController]
[Route("[controller]")]
public class WeatherForecastController : ControllerBase
{
    //Instance AWS SQS
    private readonly IEventBus eventBus;

    public WeatherForecastController(IEventBus eventBus)
    {
        this.eventBus = eventBus;
    }
}
```

Caso você queira conferir outros exemplos de utilização da biblioteca com mais detalhes, você pode clicar [aqui](https://github.com/juniortads/opentracing-sqs-csharp/tree/master/src/examples/). Caso queira acompanhar sua evolução ou mesmo enviar sugestões de evolução, acesse o repositório no [GitHub](https://github.com/juniortads/opentracing-sqs-csharp).

#### Como contribuir para OpenTracing?

![Opentracing](/images/opentracing-contribuir.png)

Você pode contribuir com o opentracing.io caso não encontre uma biblioteca específica para o teu cenário de aplicação. O processo é bastante simples! [Nesse link você vai encontrar o passo-a-passo para fazer suas contribuições](https://opentracing.io/get-involved/register/). Resumidamente, é preciso enviar um *merge request* para o [repositório do site](https://github.com/opentracing/opentracing.io) e aguardar a aprovação.

#### Conclusão

A depuração de sistemas complexos, mesmo com as ferramentas de ponta, é um grande desafio. Como a adoção de microservices vem se tornando cada vez mais ampla, a observabilidade passa a ser, cada vez mais, parte fundamental desse modelo.

*Distributed tracing* é uma ferramenta poderosa na visualização dessas transações, ajudando os times a entender falhas na comunicação entre os serviços. Agregar as interações feitas com serviço de enfileiramento de mensagens e tópicos coloca o rastreamento dessas transações em outro nível, não deixando "buracos" nas visualizações e auxiliando às equipes na construção de uma arquitetura mais eficiente e resiliente a falhas.

#### Referências Bibliográficas

- [DDOS: Taming Nondeterminism in Distributed Systems](https://sampa.cs.washington.edu/new/papers/asplos021-hunt.pdf)
- [Sagas](https://microservices.io/patterns/data/saga.html)
- [Microservices.io](https://microservices.io/)
- [What is Distributed Tracing?](https://opentracing.io/docs/overview/what-is-tracing/)
- [OpenTracing and Containers: Depth, Breadth, and the Future of Tracing](https://www.youtube.com/watch?v=n8mUiLIXkto)
- [Distributed Tracing - we've been doing it wrong](https://copyconstruct.medium.com/distributed-tracing-weve-been-doing-it-wrong-39fc92a857df)
- [Architecture - Jaeger documentation](https://www.jaegertracing.io/docs/1.17/architecture/)
- [Observability, or Knowing What Your Microservices Are Doing](https://dzone.com/articles/observability-or-quotknowing-what-your-microservic)


