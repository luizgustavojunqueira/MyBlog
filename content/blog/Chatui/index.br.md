+++
date = '2026-02-06T22:35:41-04:00'
draft = false
title = 'Chatui'
+++

Hoje vou falar sobre o Chatui, um projeto que desenvolvi esses dias para praticar um pouco de Golang e tem sido um aprendizado muito interessante. O Chatui é um chat utilizando websockets com uma TUI (Terminal User Interface) para a interface do usuário.

<!--more-->

# O que é o Chatui?

Eu queria tirar a ferrugem de Golang e, com a popularidade das TUIs, achei interessante desenvolver um chat em modo texto. Com isso, revisei goroutines, canais e pratiquei comunicação em tempo real via websockets, além de aprender melhor a biblioteca Bubble Tea, bem popular para TUI em Go.

# Estrutura do Projeto

```
chatui/
├── cmd/
│   ├── server/
│   │   └── main.go
│   └── client/
│       └── main.go
├── internal/
│   ├── server/
│   │   └── server.go
│   └── client/
│       ├── client.go
│       ├── cmd.go
│       ├── model.go
│       ├── update.go
│       └── view.go
├── protocol/
│   └── message.go
```

- `cmd`: entradas do servidor e do cliente.
- `internal/server`: lógica do servidor (hub, registro, desconexão, broadcast).
- `internal/client`: TUI com Bubble Tea (rede, update, view separados).
- `protocol`: definição das mensagens trocadas.

Envelope das mensagens:

```go
const (
    TypeChatMessage    MessageType = "chat_message"
    TypeLoginResponse  MessageType = "login_response"
    TypeUserListUpdate MessageType = "user_list_update"
    TypeLoginRequest   MessageType = "login_request"
)

type Envelope struct {
    Type MessageType     `json:"type"`
    Data json.RawMessage `json:"data"`
}
```

# Como funciona

O servidor mantém um Hub com canais para registrar/desconectar clientes e encaminhar mensagens (broadcast ou destino específico). O cliente TUI:

1. Abre tela de login para digitar usuário.
2. Envia login e espera resposta.
3. Se sucesso, mostra a tela principal com lista de chats e mensagens.
4. Envia mensagens e recebe em tempo real.
5. Alterna chats; ao focar um chat, vê o histórico e zera notificações não lidas.

# Perrengues de TUI (o que parece simples não é)

- **Fundo sólido sem quebrar cursor**: tirar bordas padrão, preencher fundos de textarea e mensagens; evitar sequências ANSI extras que cancelam o blink do cursor.
- **Blink e foco visíveis**: blink mais rápido, destaque da seleção no sidebar piscando quando focado, e hint de atalho no título ("(Tab: focus)").
- **Largura e preenchimento**: sidebar ocupa toda a largura prevista, textos centralizados, e textarea preenche todo o fundo.
- **Separação de responsabilidades**: o monolito virou `model.go` (estado/tipos/init), `view.go` (render/layout), `update.go` (mensagens/teclas), `cmd.go` (efeitos/IO/tempo). Ficou bem mais fácil de ajustar estilo e lógica sem quebrar o resto.

# Funcionalidades atuais

- Login com checagem de nome único e válido.
- Lista de chats (ALL + privados).
- Contador de não lidas por chat; ao focar, zera o contador.
- Envio e recepção em tempo real.
- Atalhos: Tab alterna foco (sidebar/chat), Enter envia, `/quit` sai.

No vídeo abaixo, é possível ver o chat em ação, com três clientes conectados e trocando mensagens, apresentando todas as funcionalidades descritas acima:

<video controls>
  <source src="https://luizgustavojunqueira-personalblog.s3.us-east-1.amazonaws.com/Chatui/screenrecording-2026-02-06_23-08-02.mp4" type="video/mp4">
  Seu navegador não suporta o elemento de vídeo.
</video>

# Possíveis incrementos

- Persistir histórico em disco.
- Mostrar timestamps e indicator de "digitando...".
- Suporte a emojis.
- Temas claro/escuro.
- Scrollback com busca.

# Conclusão

O Chatui é simples, mas rendeu bastante aprendizado: websockets em Go, TUIs com Bubble Tea, e várias sutilezas de UI em terminal (um ANSI fora do lugar e o cursor para de piscar). O código está no [github](https://github.com/luizgustavojunqueira/chatui); contribuições são bem-vindas.
