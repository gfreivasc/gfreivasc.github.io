---
layout: post
title: Scraping fácil com Python e Scrapy
date: 2016-12-29 19:00:00+0300
categories: python pt-BR
related_image: "https://cdn-images-1.medium.com/max/2000/1*lhDA4o6GyF65IZPCox_LbA.jpeg"
---

Porque spiders conseguem tudo que você quer, e muito mais

<figure class="align-center">
  <img src="https://cdn-images-1.medium.com/max/2000/1*lhDA4o6GyF65IZPCox_LbA.jpeg" alt="">
  <figcaption>Sim, eu peguei a primeira imagem relacionada que tinha no Google imagens</figcaption>
</figure>

### Scraping é tudo de bom

A internet é uma fonte infindável de conteúdo. Tornou-se comum que desenvolvedores quisessem buscar parte dessas informações em páginas alheias, organizar tudo de forma *estruturada* e dispor através de serviços e apps com *diversas* finalidades. Para facilitar o trabalho dos desenvolvedores, algumas páginas passaram a dispor do que chamamos de API.

APIs, contudo, são limitadas. Além de depender da boa vontade da equipe de desenvolvimento para que haja uma API em primeiro lugar, também depende de ainda mais boa vontade para que ela esteja 100% atualizada (estilo bomba patch).

Para tapar esse buraco, podemos fazer muita coisa. Uma das soluções que tem feito a cabeça da garotada nos verões mundo afora é o que chamamos de [*Web Scraping*](https://en.wikipedia.org/wiki/Web_scraping). A técnica consiste simplesmente em fazer leitura do que está disposto na página e guardar de forma estruturada. Páginas costumam conter tudo o que gostaríamos de saber, inclusive dados que sequer são fornecidos por APIs*.

### Ok, mas como eu faço isso?

Muito simples, muito fácil. Tudo que você precisa saber, de começo, é que, na prática, você estará cavucando o código fonte HTML de uma página. Sim, tem muita informação, muito mais do que o que queremos de fato. Como separar os itens que você quer do resto? Por *Seletores.*

Seletores são aquilo que usamos quando queremos atribuir um estilo específico a um elemento via [CSS](https://www.w3.org/TR/CSS21/selector.html%23id-selectors), ou um comportamento específico via jQuery. É simples assim separar o que queremos do resto, *selecionando*.

Digamos que queremos saber o título e o texto da seguinte página (chamemos de *index.html*)

```html
<html>
  <head>
    <title>My Page</title>
  </head>
  <body>
    <p>Text</p>
  </body>
</html>
```

Primeiro, precisaríamos enviar uma requisição que nos retornaria este HTML, depois seria feito a seleção das informações de interesse, que chamamos de *parsing.*

```python
response = RequestHtml("index.html")
title = response.select("css", "title::text")
text = response.select("css", "p::text")
```

Perceba que no método *select* eu especifiquei CSS. Isto por que há outra maneira de selecionar informações, chamada [XPath](http://www.w3schools.com/xml/xpath_intro.asp). A gente vai ver isso melhor lá pra frente.

É basicamente só isso. Bem simples a ideia, mas na prática precisamos de uma ferramenta que seja capaz de fazer o trabalho de requisição do HTML, e outra que seja capaz de manusear a resposta e fazer as seleções. Ou podemos usar uma ferramenta que resolve ambas as coisas para nós, *por que não*?

### [Scrapy](https://scrapy.org/)

A melhor ferramenta para o assunto, segundo *eu mesmo*. Ela oferece bastante poder para lidar com várias pedras que aparecem no caminho de quem tem que lidar com *spiders*.

Spider é como chamamos o código que será responsável por fazer as requisições, o parsing e por nos retornar os dados que queremos. Scrapy oferece uma base para que a gente possa montar uma spider com pouco esforço.

Primeiramente, instale o Scrapy (de preferencia em uma virtualenv).

```sh
$ pip install scrapy
```

Então, podemos criar um arquivo, digamos minimal.py

```python
import scrapy


class MinimalSpider(scrapy.Spider):
    name = 'minimal'
    start_urls = ['http://gabrielfv.com/']

    def parse(self, response):
        data = {}
        data['title'] = response.css('title::text').extract_first()
        data['text'] = response.css('p::text').extract_first()
        return data
```

Rode o código com o comando

```sh
$ scrapy runspider minimal.py -t json -o data.json
```

Gerará um arquivo *json* que terá o título da página e o texto contido no *primeiro elemento* <p> que houver. Perceba que foi necessário, além de selecionar os dados, *extraí-los*. Isto por que o Scrapy trata uma seleção como uma classe específica, [SelectorList](https://doc.scrapy.org/en/latest/topics/selectors.html), que nos permite realizar operações como continuar fazendo seleções, em cadeia. Extrair significa transformar o que foi selecionado em uma string de Python.

Perceba que uma seleção é de fato como uma lista. Uma mesma *query* pode nos retornar vários elementos diferentes. Mesmo que retorne apenas um elemento, a seleção continua sendo uma lista. Para que a gente não tenha que indexar a lista sempre que quiser extrair o primeiro (e muitas vezes único) elemento, Scrapy oferece o método extract_first().

## Indo um pouco além

<figure class="align-center">
  <img src="https://cdn-images-1.medium.com/max/2000/1*2jX9gjaHqy0L6FyaRk3sUA.jpeg" alt="">
  <figcaption>Dessa vez não foi a primeira imagem do Google Imagens</figcaption>
</figure>

Agora que você conseguiu programar uma spider simples deve estar querendo monitorar o mundo inteiro do topo do seu computador, mas vamos com calma. Ainda há mais a se entender acerca de Scrapy antes de avançar em tarefas mais ardilosas.

### Um breve adendo

Primeiramente, o código acima não segue o que recomendaria qualquer um que tenha uma certa intimidade com a framework. Nós utilizamos um dicionário para receber os dados que capturamos da página e não é uma boa ideia. Um dos propósitos chave de scraping é a organização dos dados de forma estruturada, para que possam ser facilmente acessados por outros serviços. E estrutura é algo que falta a dicionários. A gente até pode utilizá-los, como no exemplo, mas *não devemos*.
> “O que eu faço, então?” — Você, agora

Você pode ter pensado em usar classes. É mais ou menos isso mesmo. O que vamos fazer é *herdar* uma classe específica que nos é oferecida. Nos iremos definir um Item que deverá ter os campos (tipo Field) esperados definidos de forma explícita. Cada campo poderá conter, inclusive, metadados.

```python
import scrapy


class PageLinksItem(scrapy.Item):
    title = scrapy.Field()
    links = scrapy.Field()
```

Podemos, então usar esse item para guardar dados de uma spider. Vejamos uma que captura todos os links de um site específico.

```python
import scrapy


class PageLinkItem(scrapy.Item):
    title = scrapy.Field()
    links = scrapy.Field()


class LinkReaderSpider(scrapy.Spider):
    name = 'link_reader'
    start_urls = ['http://gabrielfv.com']

    def parse(self, response):
        data = PageLinkItem()
        data['title'] = response.css('title::text').extract_first()

        links = response.css('a')
        data['links'] = [
            link.xpath('./@href').extract_first() for link in links
        ]

        return data
```

Rode com

```sh
$ scrapy runspider link_reader.py -t json -o data.json
```

Sim, um dicionário faria o trabalho. Mas o problema não é em casos pequenos como este, mas quando queremos as mesmas informações por diversas spiders. O risco de que outro desenvolvedor referencie ‘urls’ ao invés de ‘links’ pro mesmo tipo de dado é real, além da possibilidade de que ocorram *typos*. Através de itens, estamos blindados deste tipo de problema, além de podermos ter controle sobre coisas como o tipo de dados ou a *serialização* de cada campo.

### As vezes o método *parse* não é suficiente

O método parse que utilizamos até agora foi responsável por todo o trabalho que tivemos, mas ele nem sempre será capaz de fazer tudo sozinho.

Imagine uma loja um e-commerce que contenha páginas de categorias de produtos diferentes, cada página tem produtos que você quer coletar preços, mas o método parse é limitado, e trabalha apenas em cima das páginas especificadas na variável start_urls da spider. Pra isso precisamos definir um outro método, que será chamado para avaliar a página de cada categoria.

O que acontece de fato é que vamos definir um *callback*, que é chamado pela engine do Scrapy após baixar o conteúdo de uma página. Toda requisição, quando feita, recebe um callback que fará um parsing. Se nenhum é definido, o padrão é chamado. No caso, o método parse é o callback padrão.

Requisições são feitas pela classe do Scrapy [Request](https://doc.scrapy.org/en/latest/topics/request-response.html#request-objects), que deve ser inicializada com a URL da página que será lida. A gente pode definir, ao inicializar, qual método será o callback da requisição, e é aí que a gente ataca.

```python
import scrapy


class StoreSpider(scrapy.Spider):
    # ...
    def parse(self, response):
        categories = response.xpath("...")
        for c in categories:
            url_category = c.xpath("./.../@href").extract_first()
            yield scrapy.Request(
                url_category, callback=self.parse_category
            )

    def parse_category(scrapy.Spider):
        products = response.xpath("...")
        ...
```

A gente pode tentar modificar nossa última spider para ler o título das páginas para as quais ela tem um link ao invés da URL:

```python
import scrapy


class PageItem(scrapy.Item):
    title = scrapy.Field()


class LinkReaderSpider(scrapy.Spider):
    name = 'link_follower'
    start_urls = ['http://gabrielfv.com']

    def parse(self, response):
        data = PageItem()
        data['title'] = response.css('title::text').extract_first()
        yield data

        links = response.css('a')
        for link in links:
            yield scrapy.Request(
                link.xpath('./@href').extract_first(),
                callback=self.parse_links
            )

    def parse_links(self, response):
        data = PageItem()
        data['title'] = response.css('title::text').extract_first()
        return data
```

Execute e veja funcionando (deve dar um erro por haver um link *mailto* na página. Apenas ignore).

## Agora já dá pra fazer algo sério, né?

Dá sim, o que não significa que tem muita coisa interessante ainda que pode salvar seu couro cabeludo de dedos confusos. Não abordarei neste texto outros tópicos como *Pipelines, Link Extractors*, configurar o scrapy pra integrar projetos maiores e outras coisas úteis como integração com banco de dados e tarefas periódicas, mas pretendo entrar nestes assuntos em outra hora.

Por hora, é importante saber que o Scrapy vem com o modo *shell,* que é ***extremamente*** útil. Quando você estiver lidando com uma página extensa da vida real, vai perceber que selecionar exatamente o que quer pode ser uma tarefa complicada. O shell serve, principalmente, pra nos auxiliar nessa tarefa. Você pode testar as queries de forma bem simples.

```sh
$ scrapy shell https://google.com
```

Carregará a página do google e lhe colocará na shell do Python como se você estivesse no começo do método parse, dando acesso à variável response que contém informações da página que você definiu. No caso, a página é [o próprio Google](https://google.com):

```sh
>>> response
<200 [https://www.google.com.br/?gfe_rd=cr&ei=_VtkWMfDHKrL8gfdsoKwBw](https://www.google.com.br/?gfe_rd=cr&ei=_VtkWMfDHKrL8gfdsoKwBw)>
```

Se tentarmos acessar algo por seletores

```sh
>>> response.xpath('//input').extract_first()
u'<input name="ie" value="ISO-8859-1" type="hidden">'
>>> response.css('img').xpath('./@src').extract_first()
u'/textinputassistant/tia.png'
```

Sim, o shell é uma parte maravilhosa do scrapy. Um único comando e voilá! Estamos prontos para testar as queries. Pode ser que o comando dê erro, dependendo da URL que você passar pra shell. Caso isso aconteça, passe a URL entre aspas, por exemplo:

```sh
$ scrapy shell "https://google.com"
```

E o shell deve funcionar.
