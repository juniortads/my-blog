+++
date = "2017-04-24"
title = "Ecossistema Elasticsearch [pt-BR]"
math = "true"

+++

Ajudando usuários e desenvolvedores a pesquisar e analisar os seus dados.

![elastic](/images/elastic-1.png)

"Não é incomum hoje em dia que as empresam coletem dados para levantar informações sobre as preferências de compra e tendências dos consumidores. É um dilúvio de dados que cresce continuamente e rapidamente. Além disso, como se costuma dizer no Vale do Silício, Dados é o novo petróleo; por isso, ser capaz de consultar, explorar e tratar grandes volumes de dados com rapidez, é algo de interesse de vários players do mercado."

Em 2013, a quantidade total de dados no mundo foi de 4.4 ZettaBytes, e espera-se que, até 2020, este número atinja 44 ZettaBytes. Lembrando que um Zettabyte equivale a 44 trilhões de gigabytes. Com essa quantidade massiva de informação sendo gerada e armazenada, realizar qualquer tipo de consulta pode se tornar uma tarefa onerosa. É um desafio para os grandes serviços, como GitHub, NetFlix, Globo.com promoverem para seus usuários uma experiência funcional em suas consultas. Pois o principal objetivo é fazer com que o usuário final não só encontre aquilo que procura, mas que tudo aconteça de maneira rápida, quase ou senão, em real-time.

{{< youtube s1ai_LCx5UM >}}

### O case GitHub
Como satisfazer as necessidades de busca de milhões de usuários enquanto fornece, simultaneamente, informações operacionais que podem ajudar a melhorar o atendimento e a experiência do cliente? Essa é a grande questão para qual várias empresas procuram uma solução, e, foi em busca dessa resposta que o GitHub, o maior sistema de controle de versão hospedado no mundo, decidiu utilizar o Elasticsearch, uma ferramenta OpenSource para buscas, acessível através de uma API REST bem elaborada que facilita a integração com diferentes plataformas, além de ser ideal para trabalhar com Big Data, permitindo que usuários e desenvolvedores explorem dados a uma velocidade e em uma escala nunca antes possível.

Com cerca de 4 milhões de usuários técnicos e bem exigentes, o GitHub usa o Elasticsearch para indexar mais de 8 milhões de repositórios de código, incluindo mais de 2 bilhões de documentos. Um dos objetivos principais da implementação do Elasticsearch ao GitHub é indexar tudo o que está publicamente disponível no github.com e torna-lo fácil de encontrar, alavancando o Google Analytics nos dados de pesquisa durante o processo. Por exemplo, se o usuário faz uma pesquisa de texto completa ou uma pesquisa estruturada, por meio do Elasticsearch, é possível consultar mais de 130 bilhões de linhas de código em pouquíssimo tempo. Segundo o engenheiro de operações do GitHub, em depoimento no case GitHub para site elastic.co, a pesquisa é a medula do GitHub, além disso, a empresa, ao utilizar consultas Elasticsearch, consegue, se necessário, mapear de forma rápida o uso do sistema, o que ajuda também a verificar se uma conta foi roubada, sequestrada, ou mesmo se o usuário fez algo ilegal.

### Satisfação dos usuário, Back Friday e grandes players

Segundo uma pesquisa da Akamai, a satisfação dos usuários cai 16% para cada segundo a mais que o usuário espera para a página carregar carregamento de uma página. Por meio do Elasticsearch, a globo.com consegue processar 180 pesquisas por segundo, atendendo 25 de milhões de pessoas por dia em busca de milhões de vídeos, fotos e artigos. Imagine agora, por exemplo, os ganhos que um site de pode ter na Black Friday com melhorias de performance e praticidade de consultas. Não é por menos que grandes players como: GitHub, Twitter, Ebay, New York Times, NetFlix entre outros já o utilizam para suas buscas em tempo real.

![elastic](/images/elastic-2.png)

Por essas e outras, o Elasticsearch ajuda a satisfazer as necessidades de pesquisa de ambos os usuários regulares e, por meio da API Elasticsearch, desenvolvedores de aplicativos também. Isso torna a sua implementação fácil pois além dos ganhos com performance e praticidade o desenvolvimento é produtivo e adota-lo seria como ter o seu próprio Google privado.

[Postado 24 fevereiro de 2017 por Joel Júnior no blog da GFT](https://blog.gft.com/br/2017/02/24/ecossistema-elasticsearch-ajudando-usuarios-e-desenvolvedores-a-pesquisar-e-analisar-os-seus-dados/)

Happy coding!