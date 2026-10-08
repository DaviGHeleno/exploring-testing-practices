# Explorando Práticas de Teste

Neste exercício, vamos explorar práticas de teste em sistemas reais utilizando a ferramenta [TestMiner](https://andrehora.github.io/testminer).

O TestMiner permite visualizar e analisar testes de software em repositórios do GitHub, fornecendo dados sobre como os projetos organizam seus testes, como eles evoluem entre versões e quais bibliotecas de teste são utilizadas.
Explore a ferramenta antes de começar para se familiarizar com seu funcionamento.

Mais detalhes no GitHub da ferramenta: https://github.com/andrehora/testminer.

---

## Passo 1: Selecionar um repositório

Escolha um repositório real que possua testes de software.
Abaixo estão alguns links para ajudá-lo a encontrar projetos interessantes:

- Python: https://github.com/topics/python?l=python
- JavaScript: https://github.com/topics/javascript?l=javascript
- TypeScript: https://github.com/topics/typescript?l=typescript
- Java: https://github.com/topics/java?l=java

- Tópicos: [ai](https://andrehora.github.io/testminer/#topic:ai), [llm](https://andrehora.github.io/testminer/#topic:llm), [api](https://andrehora.github.io/testminer/#topic:api), [nodejs](https://andrehora.github.io/testminer/#topic:nodejs), [android](https://andrehora.github.io/testminer/#topic:android)

- Por organização: [Google](https://andrehora.github.io/testminer/#google), [Microsoft](https://andrehora.github.io/testminer/#microsoft), [Apple](https://andrehora.github.io/testminer/#apple), [Facebook](https://andrehora.github.io/testminer/#facebook), [Netflix](https://andrehora.github.io/testminer/#netflix), 
[GitHub](https://andrehora.github.io/testminer/#github), [Apache](https://andrehora.github.io/testminer/#apache), [HuggingFace](https://andrehora.github.io/testminer/#huggingface)

## Passo 2: Explorar o repositório selecionado

Busque o repositório escolhido no [TestMiner](https://andrehora.github.io/testminer) e analise os dados de teste gerados pela ferramenta.

## Passo 3: Explicar uma prática de teste

Escolha uma prática ou dado de teste relevante e explique com suas próprias palavras.

---

## Instruções de entrega

1. Faça um `fork` deste repositório (saiba mais sobre forks [aqui](https://docs.github.com/pt/pull-requests/collaborating-with-pull-requests/working-with-forks/fork-a-repo)).
2. Responda às questões abaixo diretamente neste arquivo `README.md` do seu fork. Pode adicionar imagens para enriquecer sua explicação.
3. No Moodle, submeta apenas a URL do seu fork.

---

## Respostas

Repositório: [`https://github.com/pallets/flask`](https://github.com/pallets/flask)

URL TestMiner: [`https://andrehora.github.io/testminer/#pallets/flask`](https://andrehora.github.io/testminer/#pallets/flask)

Explicação:

### O que o TestMiner mostra sobre o Flask

O Flask é um microframework web em Python, mantido pela organização Pallets. Na visão geral do TestMiner, o repositório aparece assim:

| Categoria | Arquivos |
|---|---|
| Código-fonte (`source`) | 155 |
| Testes (`test`) | 36 |
| Auxiliares de teste (`test-helper`) | 35 |
| Testes de CI (`ci-test`) | 1 (`.github/workflows/tests.yaml`) |
| Mock, e2e, snapshot, fixture, benchmark, smoke | 0 |

Na aba **Test Location**, os testes ficam quase todos numa pasta própria, `tests/`, na raiz do projeto: 26 arquivos de teste e 34 auxiliares. O resto está nos exemplos (`examples/tutorial/tests` e `examples/javascript/tests`), na documentação (`docs/testing.rst`, `docs/tutorial/tests.rst`) e em `src/flask/testing.py`.

Na aba **Test History**, o TestMiner compara três versões:

| Versão | Arquivos de teste | Auxiliares de teste |
|---|---|---|
| 0.3.1 | 5 | 5 |
| 2.0.2 | 36 | 31 |
| 3.1.3 | 36 | 35 |

A suíte cresceu muito entre as primeiras versões e a 2.x. Depois disso, o número de arquivos de teste ficou estável em 36. O que continuou crescendo foi a quantidade de arquivos auxiliares, ou seja, a estrutura que dá suporte aos testes.

Duas coisas chamam atenção. Não há nenhum arquivo de mock: o Flask testa o framework rodando aplicações de verdade, em vez de simular dependências. E a categoria auxiliar é quase do tamanho da categoria de teste. Isso leva à prática que escolhi explicar.

### Prática escolhida: fixtures compartilhadas no `conftest.py` + cliente de teste

O Flask usa o **pytest** como ferramenta de teste (grupo `tests` no `pyproject.toml`). A prática central da suíte é concentrar a preparação do ambiente em **fixtures** definidas em um único arquivo, `tests/conftest.py`.

Uma fixture é uma função que prepara algo que o teste precisa e entrega isso pronto. O teste só declara que precisa daquilo, colocando o nome da fixture como parâmetro, e o pytest faz o resto. No Flask, as principais são:

```python
@pytest.fixture
def app():
    app = Flask("flask_test", root_path=os.path.dirname(__file__))
    app.config.update(TESTING=True, SECRET_KEY="test key")
    return app

@pytest.fixture
def client(app):
    return app.test_client()
```

A fixture `app` cria uma aplicação Flask nova, já em modo de teste. A fixture `client` depende de `app` e devolve um **cliente de teste**: um "navegador falso" que faz requisições HTTP à aplicação sem precisar subir um servidor. Um teste típico fica curto e legível:

```python
def test_hello(app, client):
    @app.route("/")
    def index():
        return "Hello"

    assert client.get("/").data == b"Hello"
```

Contei as funções de teste da pasta `tests/`. São 378 no total, e 248 recebem a fixture `app` e 151 recebem `client`. Ou seja, quase todos os testes partem da mesma preparação, escrita uma única vez.

O `conftest.py` também tem duas fixtures com `autouse=True`, que rodam automaticamente em **todos** os testes, mesmo que o teste não as peça:

- `_standard_os_environ`: limpa variáveis de ambiente como `FLASK_APP` e `FLASK_DEBUG`, para que o resultado de um teste não dependa da máquina onde ele roda;
- `leak_detector`: depois de cada teste, verifica se algum "contexto de aplicação" ficou aberto. Se ficou, o teste falha. Isso impede que um teste "vaze" estado para o próximo.

### Por que essa prática é boa

1. **Isolamento:** cada teste recebe uma aplicação nova, e as fixtures automáticas garantem que nada sobra de um teste para outro. Assim os testes podem rodar em qualquer ordem e dão o mesmo resultado.
2. **Menos repetição:** a configuração fica num lugar só. Se for preciso mudar como a aplicação de teste é criada, muda-se apenas o `conftest.py`, e não centenas de testes.
3. **Testes realistas sem mocks:** com o cliente de teste, as requisições passam pelo framework inteiro (rotas, contexto, resposta). Isso explica por que o TestMiner não encontra nenhum arquivo de mock.
4. **Aplicações de apoio:** a pasta `tests/test_apps/` (responsável por boa parte dos 35 auxiliares) contém pequenas aplicações Flask completas, usadas principalmente para testar a linha de comando (`flask run`, descoberta de app etc.). A fixture `test_apps` coloca essa pasta no `sys.path` só durante o teste e depois remove os módulos importados.

Um detalhe interessante: o cliente de teste não é só uma ferramenta interna. Ele fica em `src/flask/testing.py`, dentro do próprio pacote, e por isso o TestMiner o classifica como arquivo de teste dentro de `src/`. Quem desenvolve uma aplicação com Flask usa o mesmo `app.test_client()`. O tutorial oficial (`examples/tutorial/tests/conftest.py`) ensina exatamente esse padrão de fixtures `app` e `client`. Ou seja, o Flask testa a si mesmo do jeito que recomenda que seus usuários testem as aplicações deles.

### Observação sobre a classificação automática

Como o TestMiner classifica os arquivos pelo nome e pela pasta, alguns arquivos entram como "teste" sem ser testes de fato. É o caso de `docs/testing.rst` (documentação), de `tests/templates/template_test.html` e de `.../static/css/test.css` (recursos estáticos). O `conftest.py` também é contado como teste, embora seja, na prática, um auxiliar. Por isso os números são uma boa visão geral, mas vale abrir os arquivos para confirmar o que cada um é.
