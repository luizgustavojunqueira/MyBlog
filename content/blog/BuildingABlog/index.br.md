+++
date = '2025-09-28T12:59:28-04:00'
draft = false
title = 'Construindo meu Blog'
+++

Neste post, vou falar mais profundamente sobre por que e como construí este blog.

<!--more-->

# Como Começou

Há muitos meses, comecei a assistir um YouTuber chamado **ThePrimeAgen**. De vez em quando, ele fala sobre Golang e expressa seu entusiasmo pela linguagem. Intrigado por suas percepções, decidi dar uma chance ao Golang para ver do que se tratava.

Comecei a experimentar com Golang há cerca de seis meses, resolvendo alguns problemas do [BeeCrowd](https://judge.beecrowd.com/) apenas para pegar o jeito. Logo depois, construí alguns servidores REST API simples, já que Golang é muito adequado para esse tipo de software.

Mas por que criar um blog?

Sou um grande fã de blogs de tecnologia e, como a maior parte do meu conhecimento veio da leitura deles, achei que seria legal ter o meu próprio. Como mencionei em meu [post anterior](https://blog.luizgustavojunqueira.com/post/my-blog#why-i-made-this-blog), também queria testar minhas habilidades, implantar uma aplicação além do localhost e melhorar minha escrita.

## Tecnologias

Para construir este blog, decidi usar apenas tecnologias que nunca havia usado antes, então escolhi a stack GOTTH:

- **Golang**: para servir a aplicação web
- **Templ**: para templates HTML
- **TailwindCSS**: para estilização e design bonito
- **HTMX**: para melhor interatividade e navegação
- **AlpineJS**: para animações interativas

Também usei PostgreSQL como banco de dados, embora planeje mudar para SQLite no futuro.

## Como Funciona

### O Backend

Desenvolvi o backend como um pacote para que pudesse ser reutilizado por qualquer pessoa que queira um blog simples sem muito trabalho. Atualmente, as únicas opções de customização disponíveis são para o título do blog, título da página e usuário administrador.

Planejo adicionar mais opções de customização — como esquemas de cores e ajustes de layout — no futuro, uma vez que eu atenda outras prioridades.

O pacote se chama Blogo (uma mistura de "Blog" e "Go") e requer a seguinte configuração:

```go
type BlogoConfig struct {
	BlogName   string
	Title      string
	Port       string
	DB         *sql.DB
	AuthConfig *auth.AuthConfig
	Logger     *log.Logger
	Location   *time.Location
	Queries    *repository.Queries
}
```

Você pode executar o servidor com:

```go
blog.Start()
```

Um exemplo mais detalhado está disponível no meu [repositório](https://github.com/luizgustavojunqueira/Blogo/blob/main/cmd/blog/main.go), onde você pode ver a configuração do meu blog pessoal.

#### PostHandler

O servidor consiste em um PostHandler e um AuthHandler. A parte de autenticação é direta, então vou focar no PostHandler, que compreende sete funções para gerenciar posts:

- **GetPosts**: Renderiza uma lista de cards de posts como HTML usando um template Templ.
- **ViewPost**: Renderiza o post completo.
- **DeletePost**: Deleta o post especificado.
- **Editor**: Renderiza um editor simples onde posso inserir o título do post, slug, descrição e conteúdo em Markdown (é bem básico no momento, então sem imagens).
- **Edit**: Edita um post existente.
- **CreatePost**: Gera o índice, converte o conteúdo Markdown em HTML e salva o post.
- **ParseMarkdown**: Similar ao CreatePost mas retorna o conteúdo HTML — isso é usado para recarregamento ao vivo no editor.

### O Frontend

Como mencionado anteriormente, o frontend é essencialmente HTML puro com uma pequena quantidade de JavaScript fornecida por **HTMX** e **AlpineJS**, e estilizado com **TailwindCSS**.

#### Por que Templ?

Templ permite criar componentes HTML reutilizáveis que se integram perfeitamente com código Golang. Por exemplo, considere o componente MainPage que lista os cards de posts:

```go
templ MainPage(blogname string, posts []repository.Post, authenticated bool) {
	if authenticated {
		@components.Header(blogname, []string{"New Post", "Logout"}, []string{"/editor", "/logout"})
	} else {
		@components.Header(blogname, []string{"Login"}, []string{"/login"})
	}
	<ul id="posts-list" class="min-h-screen flex flex-col items-center">
		for _, post := range posts {
			@components.PostCard(post, authenticated)
		}
	</ul>
}
```

> Nota: Os componentes Header e PostCard são definidos no pacote components.

O código Golang para renderizar a página é direto:

```go
mainPage := pages.MainPage(h.blogName, h.pagetitle, posts, authenticated)
mainPage.Render(ctx, w) // w é o http.ResponseWriter
```

#### HTMX e AlpineJS

Usei uma pequena quantidade de **HTMX** principalmente para fazer requisições HTTP — como deletar um post, lidar com login e habilitar recarregamento ao vivo no editor.

**AlpineJS** é empregado principalmente para alternar a visibilidade do índice (TOC) e para mudar o esquema de cores.

### Banco de Dados

Como esta é uma aplicação relativamente simples, escolhi não usar um ORM e, em vez disso, optei pelo SQLC — uma ferramenta que compila consultas SQL em código Go, garantindo segurança de tipos.

Para migrações de banco de dados, estou usando golang-migrate.

# Conclusão

Como você pode ver, ainda há algum trabalho a ser feito (que planejo abordar algum dia, espero), mas por enquanto o blog atende minhas necessidades.

Em relação às tecnologias, realmente gostei de trabalhar com elas. Foi uma ótima experiência desenvolver esta aplicação usando a stack **GOTTH**, já que todas as ferramentas têm comunidades solidárias e excelente documentação.

Gostei tanto que estou planejando construir uma aplicação web simples para organizar finanças pessoais (por exemplo, rastrear despesas mensais), já que não encontrei um bom aplicativo gratuito disponível. Talvez algum dia eu escreva um post sobre isso também.

Até lá, obrigado por dedicar seu tempo para ler isto.

**_Confira meu [Portfólio](https://portfolio.luizgustavojunqueira.com) também. Em breve haverá um post sobre ele também._**
