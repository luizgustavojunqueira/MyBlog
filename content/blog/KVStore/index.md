+++
date = '2025-09-28T13:14:54-04:00'
draft = false
title = 'KVStore'
+++

Desvendando Paxos na Prática: Como Construímos um KV-Store Distribuído

<!--more-->

# A Proposta

Sistemas distribuídos são a espinha dorsal de muitas aplicações modernas, mas gerenciar a consistência e a tolerância a falhas neles é um desafio. Neste post, vou compartilhar nossa jornada no desenvolvimento de um Key-Value Store distribuído para uma disciplina de faculdade, utilizando o complexo, mas robusto, algoritmo de consenso Paxos e a linguagem Go.

# O que é um algoritmo de Consenso Distribuído?

Para dar um pouco de contexto, primeiro vamos definir um Sistema Distribuído.

Sistemas distribuídos são caracterizados por um conjunto de computadores independentes que se apresentam ao usuário como um sistema único e coerente. Esses sistemas são projetados para alcançar escalabilidade, disponibilidade e tolerância a falhas, características essenciais em aplicações modernas como bancos de dados distribuídos, sistemas de arquivos e plataformas de computação em nuvem.

Com isso, uma das maiores dificuldades nesses ambientes distribuídos é a coordenação entre os nós participantes da aplicação, especialmente quando é necessário garantir consistência entre estados replicados, em cenários onde falhas (nós indisponíveis, problemas de rede) podem ocorrer.

Diante desse problema, foram desenvolvidos os algoritmos de consenso distribuído, como o Paxos ou o Raft, que servem para coordenar os nós e assegurar um acordo sobre valores ou operações, mesmo em condições adversas.

## Algoritmo Paxos

O algoritmo Paxos, proposto por Leslie Lamport, é uma solução robusta para o problema do consenso em sistemas distribuídos assíncronos com falhas. Ele resolve o problema do consenso garantindo que um conjunto de nós chegue a um acordo sobre um único valor, mesmo que até metade dos nós da rede falhem. Para isso, o Paxos opera em três papéis principais: Proposer (propõe valores), Acceptor (aceita ou rejeita propostas) e Learner (aprende o valor acordado). É um algoritmo complexo, mas sua confiabilidade o torna ideal para sistemas que exigem maior consistência.

Para entender como o Paxos garante esse consenso, vamos detalhar suas duas fases principais de operação:

- **Fase 1, de preparação**:
    - **1a. Preparar**: Um Proponente (Proposer) cria uma mensagem, chamada de “Prepare”. Essa mensagem é identificada por um número único **N**, que deve ser maior que qualquer número usado anteriormente em uma mensagem. Essa mensagem é enviada para um Quórum de Aceitadores (Acceptors).
    - **1b. Promessa**: Ao receber uma mensagem, um Aceitador verifica se **N** é maior do que qualquer número de proposta que ele já tenha prometido aceitar. Caso seja, ele responde com uma Promise, comprometendo-se a não aceitar propostas com número menor que **N** e informando a proposta de maior número, **N'** que ele já aceitou, juntamente com seu valor, **V'**. Se **N** não for maior, apenas ignora ou responde com negação.

- **Fase 2, de aceitação**:
    - **2a. Aceitar**: Se o Proponente receber Promises de um Quórum de Aceitadores, ele escolhe um valor **V** para sua proposta. Se algum Aceitador informou ter aceitado uma proposta anterior, o Proponente deve escolher o valor **V'** da proposta com o maior número **N'**. Caso contrário, pode escolher um valor novo. Então, o Proponente envia uma mensagem de Accept(**N**, **V**) para o Quórum de Aceitadores.
    - **2b. Aceito**: Ao receber uma mensagem Accept(**N**, **V**), um Aceitador verifica se já prometeu não aceitar propostas com um número menor que **N**. Se não houver tal promessa, ele aceita a proposta, registra o valor **V** como aceito e envia uma mensagem Accepted(**N**, **V**) para os Aprendizes (Learners). Se já houver uma promessa para um **N** maior, apenas ignora.

Em Paxos, o consenso é alcançado quando a maioria dos Aceitadores concorda com um mesmo número de identificador de proposta. Como cada identificador é único para um Proponente e um único valor é associado a esse identificador, garantir que a maioria aceite o mesmo ID de proposta implica que eles concordarão sobre o mesmo valor.

No entanto, todo esse processo é utilizado para escolher apenas um valor, em um cenário real, teríamos um fluxo contínuo de valores acordados atuando como comandos para uma máquina de estados distribuídos. No entando, se cada comando for resultado de uma instância de Paxos, haveria uma sobrecarga significativa na rede, visto que para cada comando seriam enviadas 2 mensagens para todos os nós da rede (1a e 2a) e um nó teria que receber 2 mensagens de cada nó da rede (1b e 2b).

Para mitigar isso, uma otimização crucial é a eleição de um Líder, que simplifica a Fase 1 para decisões subsequentes, reduzindo o tráfego de rede.

# O que é um KV-Store?

Resumidamente, um Key-Value store é um tipo de banco de dados mais simples e eficiente que armazena os dados como pares chave-valor. Tem maior desempenho para leitura e escrita visto que não tem um esquema rígido nem relações entre valores armazenados. É muito utilizado em sistemas distribuídos e para cache.

# Um KV-Store com Paxos

Certo, mas como juntar as duas coisas?

Como dito anteriormente, em um sistema real utilizando paxos, teríamos um fluxo contínuo de comandos acordados aplicados em um estado distribuído. Assim, podemos entender o estado do KV-store como o resultado da aplicação em ordem de todos os comandos decididos pelas rodadas de Paxos.

Para esse projeto definimos os seguintes comandos:

- **SET** key value
- **DELETE** key

Exemplo:

Se tivermos os seguintes 2 comandos decididos:

- **SET** chave1 valor1
- **SET** chave2 valor2

O estado seria
{ chave1: valor1, chave2: valor2 }

Caso fizessemos **DELETE** chave1, teríamos: { chave2: valor2 }

# Implementação

Para a implementação desse projeto, utilizamos Golang como linguagem principal, visto que ter um suporte muito bom para concorrência e comunicação distribuída, facilitando o desenvolvimento das funcionalidades necessárias. Já para a comunicação entre os nós, utilizamos gRPC.

## gRPC e protobuf

A comunicação do gRPC necessita de uma definição protobuf das funções, parametros e respostas implementadas pelos nós.

Todo nó do nosso sistema implementa as seguintes funções:

```proto
service Paxos{
  rpc Prepare(PrepareRequest) returns (PrepareResponse);
  rpc Accept(AcceptRequest) returns (AcceptResponse);
  rpc ProposeLeader(ProposeLeaderRequest) returns (ProposeLeaderResponse);
}
message PrepareRequest {
  int64 proposal_id = 1; // número da proposta
  int64 slot_id = 2; // número do slot da proposta
}

enum CommandType{
  UNKNOWN = 0; // comando desconhecido
  SET = 1; // comando para definir um valor
  DELETE = 2; // comando para deletar um valor
}

message Command{
  CommandType type = 1; // tipo do comando
  string key = 2; // chave do valor a ser manipulado
  bytes value = 3; // valor a ser manipulado, se aplicável
  int64 proposal_id = 4; // número da proposta associada ao comando
}

message PrepareResponse {
  bool success = 1;
  string error_message = 2; // mensagem de erro, se houver
  int64 accepted_proposal_id = 3; // número da proposta aceita
  Command accepted_command = 4; // comando aceito, se houver
  int64 current_proposal_id = 5; // número da proposta atual
}

message AcceptRequest {
  int64 proposal_id = 1; // número da proposta
  int64 slot_id = 2; // número do slot da proposta
  Command command = 3; // comando a ser aceito
}

message AcceptResponse {
  bool success = 1; // indica se a aceitação foi bem-sucedida
  int64 current_proposal_id = 2; // número da proposta aceita
  string error_message = 3; // mensagem de erro, se houver
}
```

No caso, a função Prepare corresponde a **FASE 1** e a função Accept corresponde a **FASE 2** com PrepareRequest sendo **1a** e PrepareResponse sendo **1b**, e assim também para o accept.

## Eleição do líder

Para eleger o líder de um sistema paxos podemos utilizar o próprio paxos mas é um pouco mais complexo por causa de algumas peculiaridades.

Como um nó decide se ele deve tentar se eleger lider?

A forma mais simples que conseguimos pensar é por meio de heartbeats, onde o líder atual envia periodicamente uma mensagem de heart beat para todos os nós da rede. Caso um nó fique mais que um certo tempo sem receber um heart beat do líder, ele assume que o líder está offline e após um certo atraso aleatório (para evitar multiplos nós iniciando eleição ao mesmo tempo) ele tenta se propor como líder.

Para essa parte, utilizamos as seguintes funções RPC.

```proto
service Paxos{
  rpc ProposeLeader(ProposeLeaderRequest) returns (ProposeLeaderResponse);
  rpc SendHeartbeat(LeaderHeartbeat) returns (LeaderHeartbeatResponse);
}

message ProposeLeaderRequest{
  int64 proposal_id = 1; // Número da proposta para a eleição do líder
  string candidate_address = 2; // Endereço do candidato a líder
}

message ProposeLeaderResponse{
  bool success = 1;
  string error_message = 2; // Mensagem de erro, se houver
  int64 current_highest_leader_proposal_id = 3; // Número da proposta do líder atual
  string current_leader_address = 4; // Endereço do líder atual
  int64 highest_decided_slot_id = 5; // Slot mais alto decidido até o momento
}

message LeaderHeartbeat{
  string leader_address = 1; // Endereço do líder
  int64 current_proposal_id = 2; // Número da proposta atual do líder
  int64 highest_decided_slot_id = 3; // Slot mais alto decidido até o momento
}

message LeaderHeartbeatResponse{
  bool success = 1; // Indica se o heartbeat foi bem-sucedido
  string error_message = 2; // Mensagem de erro, se houver
  int64 known_highest_slot_id = 3; // Slot mais alto que o Acceptor/Learner conhece
}
```

Além disso, para evitar problemas que encontramos durante os testes, caso mais de 1 nó perceba a falta de líder, e começem uma eleição, caso o nó receba alguma indicação que já está acontecendo uma eleição com um ID maior que a eleição dele, seja por receber a proposta diretamente, ou por receber uma negação de outro nó contendo esse ID maior, ele aborta a própria eleição.

## Cliente

Para conseguir utilizar o KV-Store, sem ter acesso direto aos nós e uma forma de executar essa funções por eles, todos os nós também implementam a seguinte definição gRPC:

```proto
service KVStore{
  rpc Get(GetRequest) returns (GetResponse);
  rpc Set(SetRequest) returns (SetResponse);
  rpc Delete(DeleteRequest) returns (DeleteResponse);
  rpc List(ListRequest) returns (ListResponse);
  rpc ListLog(ListRequest) returns (ListLogResponse);
  rpc TryElectSelf(TryElectRequest) returns (TryElectResponse);
}

message GetRequest {
  string key = 1;
}

message GetResponse {
  string value = 1;
  bool found = 2;
  string error_message = 3;
}

message SetRequest {
  string key = 1;
  string value = 2;
}

message SetResponse {
  bool success = 1;
  string error_message = 2;
}

message DeleteRequest {
  string key = 1;
}

message DeleteResponse {
  bool success = 1;
  string error_message = 2;
}

message ListRequest{}

message ListResponse {
  repeated KeyValuePair pairs = 1;
  string error_message = 3; // Mensagem de erro, se houver
}

message KeyValuePair {
  string key = 1;
  string value = 2;
}

message ListLogResponse {
  repeated LogEntry entries = 1;
  string errorMessage = 2;
}

message LogEntry {
  int64 slot_id = 1;
  Command command = 2; // Comando associado ao log
}

message Command{
  paxos.CommandType type = 1; // tipo do comando
  string key = 2; // chave do valor a ser manipulado
  string value = 3; // valor a ser manipulado, se aplicável
  int64 proposal_id = 4; // número da proposta associada ao comando
}

message TryElectRequest {
}

message TryElectResponse {
  bool success = 1; // Indica se a eleição foi bem-sucedida
  string error_message = 2; // Mensagem de erro, se houver
}
```

Assim, é possível utilizar um cliente gRPC que faz essas chamadas para algum nó específico.

Para uma melhor interação com o usuário, foi desenvolvido também um servidor HTTP (também em Golang), que atua como cliente gRPC do serviço KVStore. Assim, acompanhado do frontend desenvolvido em React, podemos visualizar e mandar comandos para cada nó do sistema.

## Registry

Certo, mas como os nós e o servidor HTTP sabem os endereços dos nós para conseguirem fazer as requisições gRPC?

Também desenvolvemos um serviço de registry simples, também por meio de gRPC, no qual cada nó se registra ao iniciar e permite que sejam consultados os endereços de todos os nós do sistema. Além da utilização de heartbeats para conhecimento de qual nó está ativo ou não.

O registry é definido pelo seguinte protobuf:

```proto
service Registry {
  rpc Register(RegisterRequest) returns (RegisterResponse);
  rpc ListAll(google.protobuf.Empty) returns (ListResponse);
  rpc Heartbeat(HeartbeatRequest) returns (HeartbeatResponse);
}
```

## Sincronização

A parte de sincronização diz respeito a sincronização do estado dos comandos decididos entre todos os nós.

Se temos um sistema com 50 nós rodando, fazemos a inserção de alguns valores e depois 10 novos nós são conectados, eles precisam saber do histórico dos valores decididos pelos outros 50. Para isso, também temos a seguinte função:

```proto
service Paxos{
  rpc Learn(LearnRequest) returns (LearnResponse);
}

message LearnRequest{
  int64 slot_id = 1; // Número do slot que está sendo aprendido
}

message LearnResponse{
  bool decided = 1; // Indica se o slot foi decidido
  Command command = 2; // Comando associado ao slot, se decidido
}
```

Quando um nó recebe um HeartBeat, é enviado junto o número do ultimo slot decidido pelo sistema e, caso seja maior que o que o nó tem no seu estado interno, ele executa **Learn** até sincronizar todo o seu estado.

Essa sincronização também é feita durante a eleição de um novo líder caso necessário.

# Conclusão

A implementação desse projeto foi algo desafiador e que trouxe muito aprendizado sobre problemas de concorrência e sistemas distribuidos, além de possibilitar uma maior prática com Golang que é uma linguagem muito interessante para sistemas concorrentes e o protocolo de comunicação gRPC.

O código para o projeto pode ser encontrado no nosso [Repositório do GitHub](https://github.com/luizgustavojunqueira/KV-Store-Paxos?tab=readme-ov-file#key-value-store-with-paxos) junto com instruções de como executar todos os serviços envolvidos.
