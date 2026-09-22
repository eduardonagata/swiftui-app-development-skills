---
description: "Mostra as chamadas de ferramenta de um LLM na própria interface de chat em SwiftUI, como pílulas intercaladas com o texto gerado, na ordem em que aconteceram. Use quando o usuário pedir para exibir/indicar/notificar o uso de ferramentas (tool calling) na tela do chat, com Foundation Models."
name: show-chat-tool-calls
---

## Mostre as chamadas de ferramenta na interface de chat

Use esta skill quando um chat em SwiftUI com Foundation Models usa ferramentas
(`Tool`) e o usuário quer ver na tela **o quê** o modelo está fazendo, no lugar
certo: uma pílula ("Consultar categorias") entre o texto que o modelo escreveu
antes de chamar a ferramenta e o que ele escreveu depois.

Esta skill continua a `build-chat-ui`. A estrutura de lá (`ContentView`/`ChatView`,
`Conversa`, `LinhaMensagem`, `BlocoMarkdown`, `Mensagem`, a rolagem que gruda no
fim) permanece inteira; o que muda é **o que a lista contém** e **de onde ela vem**.
Leia a `build-chat-ui` primeiro.

### Instruções

Não é necessário copiar o código exatamente. O que importa é reusar a forma: a
lista da tela deixa de ser `[Mensagem]` e passa a ser uma linha do tempo de itens
heterogêneos, remontada a cada snapshot a partir do transcript da sessão.

### 1. O conceito: quem guarda a ordem é o transcript

Não existe callback, delegate ou notificação de "o modelo chamou uma ferramenta".
O Foundation Models é *pull*: a sessão mantém um `Transcript`, e você o lê. Um
`Transcript` é uma coleção de `Transcript.Entry`, na ordem em que as coisas
aconteceram:

| Caso | O que é |
|---|---|
| `.instructions` | as instruções que você deu ao modelo |
| `.prompt` | o que o usuário mandou |
| `.response` | **uma** resposta do modelo, com `id` e `segments` |
| `.toolCalls` | uma coleção de `Transcript.ToolCall` (`id`, `toolName`, `arguments`) |
| `.toolOutput` | o que a ferramenta devolveu (`id`, `toolName`, `segments`) |
| `.reasoning` | o raciocínio do modelo |

Três fatos fazem a intercalação funcionar sozinha, e vale entendê-los antes de
escrever qualquer código:

1. **Uma rodada de ferramenta divide a resposta em duas.** Quando o modelo escreve
   um texto, chama uma ferramenta e volta a escrever, o framework pede uma resposta
   nova ao modelo depois de anexar a saída da ferramenta. São duas entradas
   `.response` distintas, com `id` distintos — e é isso que permite empilhar
   *texto → pílula → texto*, em vez de tudo colapsar numa bolha só.
2. **A entrada `.toolCalls` abre antes de a ferramenta rodar**, assim que o modelo
   começa a escrever os argumentos. A pílula pode aparecer imediatamente, girando.
3. **`ToolOutput.id` repete o `ToolCall.id`.** É esse par que diz "esta chamada
   terminou", e portanto é ele que troca o `ProgressView` pelo `checkmark`.

Durante o streaming, cada snapshot do stream carrega, além de `.content`, a fatia
do transcript produzida por aquela resposta:

```swift
for try await parcial in sessao.streamResponse(to: texto) {
    let entradas: ArraySlice<Transcript.Entry> = parcial.transcriptEntries
}
```

`transcriptEntries` é uma `ArraySlice` — uma **janela** sobre o transcript da
sessão, não uma lista solta. Requer macOS 27 / iOS 27. Em alvos anteriores, o
caminho equivalente é ler `sessao.transcript`, que é observável, e desenhar a
partir dele; a tradução das entradas (seção 3) é a mesma.

### 2. O modelo: uma linha do tempo, não uma lista de mensagens

A tela passa a mostrar dois tipos de coisa, então o `ForEach` precisa de um item
que seja uma ou outra. `ItemConversa.swift`:

```swift
import Foundation

/// Um item da linha do tempo da conversa.
enum ItemConversa: Identifiable {
    case mensagem(Mensagem)
    case ferramenta(UsoDeFerramenta)

    var id: String {
        switch self {
        case .mensagem(let mensagem): "mensagem-\(mensagem.id)"
        case .ferramenta(let uso): "ferramenta-\(uso.id)"
        }
    }
}

/// Uma chamada de ferramenta como a tela a mostra.
struct UsoDeFerramenta: Identifiable {
    /// O id da `Transcript.ToolCall`. O `Transcript.ToolOutput` correspondente
    /// repete esse id, e é assim que a chamada sabe que terminou.
    let id: String
    let nome: String
    var concluida = false
}
```

O `id` do item é um `String` com prefixo. O prefixo é o que garante que uma
mensagem e uma pílula nunca disputem a mesma identidade no `ForEach`, ainda que
os ids de origem coincidissem.

Isso obriga uma mudança pequena na `Mensagem` da `build-chat-ui`: o `id` deixa de
ser `UUID` e passa a ser `String`, porque a identidade de uma mensagem do
assistente agora **vem de fora** — é o `id` da entrada `.response`. Manter esse id
é o que faz o SwiftUI reaproveitar a view entre snapshots em vez de recriá-la.

```swift
struct Mensagem: Identifiable {
    let id: String
    let autor: Autor
    let texto: String

    init(id: String = UUID().uuidString, autor: Autor, texto: String) { ... }
}
```

Consequência em cascata, fácil de esquecer: `ScrollPosition(idType: UUID.self)`
vira `ScrollPosition(idType: String.self)`.

### 3. A `Conversa`: remontar a linha do tempo a cada snapshot

Aqui está o coração da skill. Em vez de localizar uma mensagem pelo `id` e
sobrescrevê-la (como na `build-chat-ui`), substituímos **tudo que esta resposta
produziu** pela tradução do transcript:

```swift
func enviar(_ texto: String) {
    guard !respondendo else { return }
    respondendo = true

    tarefa = Task {
        defer { respondendo = false }
        itens.append(.mensagem(Mensagem(autor: .usuario, texto: texto)))
        // Tudo que esta resposta produzir — texto e ferramentas — ocupa a
        // lista a partir daqui, e é remontado a cada passo do stream.
        let inicio = itens.count

        do {
            for try await parcial in sessao.streamResponse(to: texto) {
                itens.replaceSubrange(
                    inicio...,
                    with: Self.linhaDoTempo(de: parcial.transcriptEntries)
                )
            }
        } catch is CancellationError {
            // O usuário parou: o que já apareceu fica na tela.
        } catch {
            itens.append(.mensagem(Mensagem(autor: .assistente, texto: "erro: \(error)")))
        }
    }
}
```

`inicio` é marcado depois de anexar a mensagem do usuário, e `replaceSubrange`
recorta dali até o fim. Assim as rodadas anteriores da conversa ficam intactas e a
rodada em andamento é refeita por inteiro a cada snapshot. **Não há estado
incremental**: nada para dessincronizar, nada para "esquecer" de atualizar. É a
mesma ideia da `build-chat-ui`, em que cada snapshot traz o texto inteiro em vez de
um delta — só que agora aplicada à lista, e não a uma string.

A tradução:

```swift
/// Traduz as entradas do transcript na linha do tempo que a tela mostra.
private static func linhaDoTempo(
    de entradas: some Sequence<Transcript.Entry>
) -> [ItemConversa] {
    var itens: [ItemConversa] = []
    var concluidas: Set<String> = []

    for entrada in entradas {
        switch entrada {
        case .response(let resposta):
            let texto = Self.texto(de: resposta.segments)
            // Uma resposta recém-aberta ainda não tem texto: sem isto ela
            // abriria um vão no VStack antes do primeiro token chegar.
            if !texto.isEmpty {
                itens.append(
                    .mensagem(Mensagem(id: resposta.id, autor: .assistente, texto: texto))
                )
            }

        case .toolCalls(let chamadas):
            for chamada in chamadas {
                itens.append(
                    .ferramenta(UsoDeFerramenta(id: chamada.id, nome: chamada.toolName))
                )
            }

        case .toolOutput(let saida):
            concluidas.insert(saida.id)

        // O prompt já entrou na lista quando o usuário enviou; instruções e
        // raciocínio não vão para a tela.
        case .instructions, .prompt, .reasoning:
            break

        @unknown default:
            break
        }
    }

    // A saída de uma ferramenta vem depois da chamada, então a conclusão só
    // pode ser marcada com a lista inteira já percorrida.
    return itens.map { item in
        guard case .ferramenta(var uso) = item, concluidas.contains(uso.id) else { return item }
        uso.concluida = true
        return .ferramenta(uso)
    }
}

private static func texto(de segmentos: [Transcript.Segment]) -> String {
    segmentos
        .compactMap { segmento -> String? in
            guard case .text(let trecho) = segmento else { return nil }
            return trecho.content
        }
        .joined()
}
```

Quatro decisões dentro dessa função, todas por um motivo:

- **`.toolOutput` não gera item.** Ele não é uma coisa na tela, é uma *mudança de
  estado* de uma pílula que já está lá. Por isso vai para um `Set` de ids, e a
  marcação acontece no `map` do fim — a saída sempre vem depois da chamada, então
  não é possível decidir isso na primeira passada.
- **Texto vazio não entra.** O framework abre a entrada `.response` antes do
  primeiro token; uma mensagem vazia não desenha nada, mas ainda consome o
  `spacing` do `VStack` e abre um vão visível.
- **`.prompt` é ignorado.** A mensagem do usuário já foi anexada por `enviar`. Se
  a fatia do transcript incluir o prompt, ignorá-lo evita duplicá-lo — e se não
  incluir, o `case` simplesmente nunca é atingido. Barato dos dois lados.
- **Só segmentos `.text`.** `.structure` e `.attachment` existem para saída
  estruturada e anexos; num chat de texto eles não têm o que renderizar.

### 4. A view: um `switch` dentro do `ForEach`

No `ChatView` da `build-chat-ui`, muda só o corpo do `ForEach`:

```swift
ForEach(conversa.itens) { item in
    switch item {
    case .mensagem(let mensagem):
        LinhaMensagem(mensagem: mensagem)
    case .ferramenta(let uso):
        PilulaFerramenta(uso: uso)
    }
}
```

E muda a chave da rolagem automática. A chave antiga
(`.onChange(of: conversa.mensagens.last?.texto)`) não vê a pílula concluir, e
concluir muda a altura do conteúdo. Uma computada cobre os dois casos que mexem na
altura **sem** mexer na contagem de itens:

```swift
/// O que muda no último item a cada passo do stream, sem que a contagem de
/// itens mude: o texto que cresce, ou a pílula que conclui.
private var fim: String {
    switch conversa.itens.last {
    case .mensagem(let mensagem): mensagem.texto
    case .ferramenta(let uso): "\(uso.id)-\(uso.concluida)"
    case nil: ""
    }
}
```

`.onChange(of: fim)` gruda no fundo durante a geração; o
`.onChange(of: conversa.itens.count)` continua religando `usuarioRolou`, igual à
`build-chat-ui`.

### 5. A pílula

```swift
import SwiftUI

/// A pílula que mostra uma chamada de ferramenta no meio da conversa.
struct PilulaFerramenta: View {
    let uso: UsoDeFerramenta

    var body: some View {
        HStack(spacing: 6) {
            if uso.concluida {
                Image(systemName: "checkmark")
            } else {
                ProgressView()
                    .controlSize(.mini)
            }
            Text(rotulo)
        }
        .font(.caption)
        .foregroundStyle(.secondary)
        .padding(.horizontal, 10)
        .padding(.vertical, 5)
        .background(.quinary, in: .capsule)
        .frame(maxWidth: .infinity, alignment: .leading)
    }

    /// O nome que o modelo usa não é para ser lido pelo usuário, então cada
    /// ferramenta tem um rótulo próprio. Uma ferramenta nova sem rótulo cai no
    /// `default` e aparece pelo nome, em vez de desaparecer da tela.
    private var rotulo: String {
        switch uso.nome {
        case "listarCategorias": "Consultar categorias"
        // ... um caso por ferramenta
        default: uso.nome
        }
    }
}
```

A pílula fica à esquerda e estreita, então convive com o alinhamento das bolhas da
`build-chat-ui` sem ajuste nenhum. O `default` é deliberado: uma ferramenta nova
aparece com o nome técnico — feio, mas visível — em vez de sumir silenciosamente.

### 6. O que verificar rodando, e os limites

- **Duas pílulas seguidas.** Se o modelo emitir duas chamadas na mesma resposta,
  as duas pílulas saem empilhadas antes de qualquer texto novo. É a ordem real de
  geração, mas parece diferente de "pílula, texto, pílula". Não é defeito.
- **Pílula girando para sempre** indica que o `id` da saída não bateu com o da
  chamada. O texto e a ordem continuam certos; só a conclusão não é detectada.
- **Argumentos na pílula** (`chamada.arguments`) chegam como JSON parcial durante
  o stream. Se quiser mostrá-los ("Registrar despesa: Mercado"), só exiba depois
  que a chamada estiver concluída, senão aparece texto truncado.
- **Erros de ferramenta.** Um erro lançado de dentro de uma `Tool` vira
  `ToolCallError` e aborta a resposta: a pílula fica inacabada e cai no `catch`.
  Falha de regra é melhor devolvida como texto ("Falha: ..."), que o modelo lê e
  corrige — e a pílula conclui normalmente.

Existe um caminho alternativo, mais simples: notificar de dentro do `call` de cada
`Tool`, que é código seu. Ele dá só "a ferramenta X rodou", sem a posição dela em
relação ao texto, e exige um canal `Sendable` se as ferramentas forem `nonisolated`.
O transcript resolve a ordenação de graça, e por isso é o caminho desta skill.

### 7. Considerações finais

Neste documento eu descrevo o meu entendimento sobre esse mecanismo. A propriedade
boa desta estrutura é a mesma da `build-chat-ui`: a tela é uma **função pura** do
transcript da sessão. Não há um segundo estado a manter em sincronia com o modelo —
a cada snapshot a rodada é redesenhada do zero, e a ordem que aparece na tela é
literalmente a ordem em que o modelo gerou as coisas.

Isso também é o que torna a estrutura extensível sem reescrita: mostrar o raciocínio
do modelo é tirar `.reasoning` do `case` que ignora, acrescentar um caso em
`ItemConversa` e um caso no `switch` do `ForEach`. Nada mais muda.

Ao término, descreva de maneira sucinta a estrutura no CLAUDE.md: a linha do tempo,
a tradução das entradas do transcript, e a convenção de acrescentar um rótulo na
pílula a cada ferramenta nova.
