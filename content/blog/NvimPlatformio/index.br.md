+++
date = '2025-10-08T18:26:58-04:00'
draft = false
title = 'NvimPlatformio'
+++

Hoje vou compartilhar um pouco do meu setup de desenvolvimento na plataforma Arduino utilizando o PlatformIO

<!--more-->

## Introdução

Bom, primeiramente, o que é o PlatformIO?

> PlatformIO é um ecossistema de desenvolvimento para IoT que inclui um IDE, um sistema de construção e uma biblioteca de gerenciamento. Ele suporta mais de 1500 placas de desenvolvimento, incluindo Arduino, ESP8266, ESP32, STM32 e muitas outras. O PlatformIO é uma alternativa ao Arduino IDE tradicional, oferecendo mais recursos e flexibilidade para desenvolvedores.

Pode ser usado como uma extensão do VSCode, ou diretamente como CLI.

Eu utilizo por preferência com relação a IDE do Arduino, visto que o em um setup com platformio, seja com VSCode ou Neovim, o ambiente de desenvolvimento é mais robusto, com mais recursos e flexibilidade, e, o mais importante para mim, a LSP funciona muito melhor do que a da IDE do Arduino.

Segundamente, por que o Neovim?

Eu utilizo o Neovim como editor de texto principal, e, por consequência, como IDE. Acredito que o Neovim é uma ferramenta poderosa e flexível, que me permite personalizar meu ambiente de desenvolvimento de acordo com minhas necessidades e me permite maior velocidade de desenvolvimento.

## Setup

Bom, antigamente eu utilizava uma configuração personalizada do Neovim que eu mesmo montei ao longo de uns meses, mas já estava se tornando bem instável e causando algumas dificuldades em alguns casos. Com isso, junto com minha migração de Arch + Hyprland para Omarchy, que já vem por padrão com LazyVim configurado, decidi utilizar o LazyVim como base para meu setup.

Anteriormente eu havia configurado o clangd para funcionar como LSP para o platformio, mas com o LazyVim, o setup ficou um pouco diferente mas mais simples.

Bom, primeiramente, para que isso funcione é necessário ter a CLI do PlatformIO instalada. Para isso, basta seguir as instruções no site oficial: https://platformio.org/install/cli

Assim, você terá acesso ao comando `pio` no terminal, que será essencial para o funcionamento do setup.

Além disso, é necessário ter a LSP `clandg` instalada, pode ser pelo `Mason` mesmo, que é o gerenciador de LSP padrão do LazyVim.

## Configuração do LazyVim

Bom, como o clangd por padrão nao vem com suporte ao PlatformIO, é necessário adicionar algumas flags para que ele funcione corretamente.

Essa configuração vai depender de qual placa de desenvolvimento você pretende utilizar, pois terá que fazer a instalação da toolchain correta para a placa.

A toolchain você pode instalar pelo menu `Platforms` do PlatformIO, ou diretamente pelo terminal com o comando:

```bash
pio platform install <nome_da_plataforma>
```


Por exemplo, para instalar a toolchain do ESP32, o comando seria:

```bash
pio platform install espressif32
```

Isso vai adicionar uma nova pasta na sua instalação do PlatformIO, que por padrão fica em `~/.platformio/packages/`, mas não necessariamente com o mesmo nome da plataforma, por exemplo, o espressif32 fica em `~/.platformio/packages/toolchain-xtensa32` e o esp8266 em `~/.platformio/packages/toolchain-xtensa`.


Bom, uma vez que temos isso, vamos para a configuração do LazyVim.

Como não consegui descobrir e não tive tempo de estudar muito como fazer a configuração usando o nvim-lspconfig, fiz um Autocmd para iniciar o clangd com as flags corretas quando um arquivo `.h` ou `.cpp` for aberto. Deixei sem nenhuma checagem se está um projeto do PlatformIO ou não, visto que uso cpp apenas para isso, e para os casos que precisei que não envolve o PlatformIO, não tive problemas com isso, o clangd funcionou normal.

Então, o LazyVim já vem com um arquivo em `~/.config/nvim/lua/config/autocmds.lua` vazio, onde eu adicionei o seguinte código:


```lua
local clangdesp_started = false

vim.api.nvim_create_user_command("StartClangdesp", function()
  if clangdesp_started then
    vim.notify("clangdesp já está rodando", vim.log.levels.INFO)
    return
  end

  local client_id = vim.lsp.start({
    name = "clangdesp",
    cmd = {
      "clangd",
      "--background-index",
      "-j=12",
      "--clang-tidy",
      "--all-scopes-completion",
      "--cross-file-rename",
      "--completion-style=detailed",
      "--header-insertion-decorators",
      "--header-insertion=iwyu",
      "--pch-storage=memory",
      "--suggest-missing-includes",
      "--query-driver=/home/seu_usuario/.platformio/packages/toolchain-xtensa-esp32/bin/xtensa-esp32-elf-gcc*,/home/luizg/.platformio/packages/toolchain-xtensa-esp32/bin/xtensa-esp32-elf-g++*,xtensa-esp32-elf-gcc*,xtensa-esp32-elf-g++*",
      "--log=verbose",
    },
    root_dir = vim.fs.dirname(vim.fs.find({ "compile_commands.json", ".clangd" }, { upward = true })[1])
      or vim.fn.getcwd(),
    single_file_support = true,
    capabilities = {
      offsetEncoding = { "utf-8", "utf-16" },
      textDocument = {
        completion = {
          completionItem = {
            commitCharactersSupport = false,
            deprecatedSupport = true,
            documentationFormat = { "markdown", "plaintext" },
            insertReplaceSupport = true,
            insertTextModeSupport = {
              valueSet = { 1 },
            },
            labelDetailsSupport = true,
            preselectSupport = false,
            resolveSupport = {
              properties = { "documentation", "detail", "additionalTextEdits", "command", "data" },
            },
            snippetSupport = true,
            tagSupport = {
              valueSet = { 1 },
            },
          },
          completionList = {
            itemDefaults = { "commitCharacters", "editRange", "insertTextFormat", "insertTextMode", "data" },
          },
          contextSupport = true,
          editsNearCursor = true,
          insertTextMode = 1,
        },
        foldingRange = {
          dynamicRegistration = false,
          lineFoldingOnly = true,
        },
      },
    },
  })
  if client_id then
    clangdesp_started = true
    vim.notify("clangdesp iniciado!", vim.log.levels.INFO)
  end
end, {})

vim.api.nvim_create_autocmd("FileType", {
  pattern = { "cpp", "h" },
  callback = function(args)
    local bufnr = args.buf

    local clangdesp_clients = vim.lsp.get_clients({ name = "clangdesp" })

    if #clangdesp_clients == 0 and not clangdesp_started then
      vim.cmd("StartClangdesp")
      vim.defer_fn(function()
        local clients = vim.lsp.get_clients({ name = "clangdesp" })
        if #clients > 0 then
          vim.lsp.buf_attach_client(bufnr, clients[1].id)
        end
      end, 500)
    elseif #clangdesp_clients > 0 then
      local client = clangdesp_clients[1]
      if not vim.lsp.buf_is_attached(bufnr, client.id) then
        vim.lsp.buf_attach_client(bufnr, client.id)
      end
    end
  end,
  desc = "Inicia clangdesp e anexa aos buffers C++/H",
})
```

É importante alterar o caminho do `--query-driver` para o caminho correto da toolchain que você instalou, caso contrário o clangd não vai funcionar corretamente com a plataforma.

Agora que temos isso, basta abrir um arquivo `.cpp` ou `.h`? Não, ainda falta uma coisa.

Para identificar as bibliotecas e includes que você adicionar pelo PlatformIO corretamente, é necessário gerar o arquivo `compile_commands.json`, que é um arquivo que contém as flags de compilação para cada arquivo do projeto.


Para isso, eu costumo adicionar o seguinte script no `Makefile` dos meus projetos PlatformIO:


```makefile
.PHONY: compiledb

ifeq ($(strip $(RUN_TESTS)),)
  EXTRA_FILTER_FLAGS :=
else
  RUN_LIST := $(subst ;, ,$(RUN_TESTS))
  EXTRA_FILTER_FLAGS := $(foreach t,$(RUN_LIST),-f "$(t)*")
endif

erase:
	@echo "Erasing the board"
	pio run -t erase --environment $(ENV)

compiledb:	
	@echo "Generating compile_commands.json"
	rm -rf compile_commands.json .pio/ .cache/ && pio run -t compiledb --environment $(ENV) 

upload_and_monitor: upload monitor

upload:
	@echo "Uploading"
	pio run -t upload --environment ${ENV}

monitor: 
	@echo "Monitoring"
	pio run -t monitor
```


Para usar, basta o comando: `ENV=<nome_do_ambiente> make compiledb`

Mesma ideia para os outros comandos, de monitor, upload e upload_and_monitor.

Pronto, agora como podemos ver abaixo, sem erros no editor e com auto complete funcionando perfeitamente.

<img src="https://luizgustavojunqueira-personalblog.s3.us-east-1.amazonaws.com/NvimPlatformio/editor.png" alt="Nvim PlatformIO" />


## Considerações finais

Bom, é isso, espero que tenha ajudado alguém com esse setup. Se tiver alguma dúvida ou sugestão, pode deixar nos comentários.

Já pesso desculpas anteriormente para os fanáticos por neovim e lua, sei que o código não está 100% otimizado e pode ter algumas melhorias, mas é o que eu consegui fazer até agora e está funcional para o meu uso.

Até porque, um ponto que acho que seria interessante mudar, mas ainda não tive tempo para testar, é mudar o query-driver com base na plataforma que está sendo usada no momento, pois da forma que está atualmente, caso esteja usando ESP32 e queira trocar para ESP8266, terá que alterar manualmente a configuração do clangd. (O make ainda irá funcionar, apenas terá milhões de erros no editor)
