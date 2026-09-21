## Crie uma interface de chat em SwiftUI

Use esta skill para criar uma interface de chat com um LLM, com streaming de respostas e renderização em markdown.

### Instruções

Esta skill estabelece a estrutura básica de uma interface de chat que eu construí e cujo funcionamento eu compreendo. Portanto, utilizá-la aumenta a compreensibilidade do projeto gerado. 

Não é necessário ser uma cópia exata do código aqui exposto, apenas reutilizar a forma e estrutura, como modelo de renderização de markdown como blocos na view, o mecanismo de "grudar" a rolagem na borda inferior do conteúdo (onde está a mensagem mais recente), etc.

### 1.1. O container

Tudo começa nessa view:

ContentView.swift:
```swift
import SwiftUI

struct ContentView: View {
    @State private var conversa = Conversa()
    @State private var rascunho = ""
    @State private var posicao = ScrollPosition(idType: UUID.self)
    @State private var usuarioRolou = false
    
    var body: some View {
        VStack(spacing: 0) {
            ScrollView {
                VStack(alignment: .leading, spacing: 8) {
                    ForEach(conversa.mensagens) { mensagem in
                        LinhaMensagem(mensagem: mensagem)
                    }
                    if conversa.respondendo {
                        ProgressView()
                            .controlSize(.small)
                            .frame(maxWidth: .infinity, alignment: .leading)
                    }
                }
                .padding()
                .scrollTargetLayout()
            }
            .scrollPosition($posicao)
            .onScrollPhaseChange { _, novaFase in
                if novaFase == .interacting {
                    usuarioRolou = true
                }
            }
            .onChange(of: conversa.mensagens.last?.texto) { _, _ in
                if !usuarioRolou {
                    posicao.scrollTo(edge: .bottom)
                }
            }
            .onChange(of: conversa.mensagens.count) { _, _ in
                usuarioRolou = false
                posicao.scrollTo(edge: .bottom)
            }

            Divider()
            
            HStack(alignment: .bottom, spacing: 8) {
                TextField("Mensagem", text: $rascunho, axis: .vertical)
                    .textFieldStyle(.plain)
                    .lineLimit(1...5)
                
                if conversa.respondendo {
                    Button("Parar", action: conversa.parar)
                        .keyboardShortcut(".", modifiers: .command)
                } else {
                    Button("Enviar", action: enviar)
                        .keyboardShortcut(.return, modifiers: .command)
                        .disabled(conversa.respondendo || rascunho.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty)
                }
                
            }
            .padding()
        }
        
    }
    
    private func enviar() {
        let texto = rascunho.trimmingCharacters(in: .whitespacesAndNewlines)
        guard !texto.isEmpty else { return }

        conversa.enviar(texto)
        rascunho = ""
    }
    
}
```

Nessa view definimos o layout do container principal: uma ScrollView com a lista de mensagens, renderizadas dentro de um VStack, e abaixo dela a barra de entrada de texto. Esse VStack é marcado com `.scrollTargetLayout()`, o que faz de cada filho dele (cada mensagem) um alvo de rolagem identificado pelo seu `id`.

Definimos `posicao`, um `@State` do tipo ScrollPosition (com `idType: UUID.self`, o mesmo tipo do `id` das mensagens), que é passado como binding (`$posicao`) para `.scrollPosition`. É por ele que comandamos a rolagem programaticamente. O ForEach percorre a lista de mensagens; como `Mensagem` conforma com Identifiable, pode ser passada diretamente ao ForEach sem informar `id:`. Dentro do ForEach renderizamos cada Mensagem dentro de uma view LinhaMensagem.

Em .onChange(of: conversa.mensagens.last?.texto), que executa a cada atualização do texto da última mensagem (ou seja, a cada snapshot parcial recebido do stream, que pode conter um ou mais tokens), verificamos se o usuário rolou manualmente, caso em que não deverá fazer o scroll automático. Se ele não rolou, realiza o scroll automático com posicao.scrollTo(edge: .bottom).

.onScrollPhaseChange detecta quando o usuário interage com a rolagem (em qualquer direção, não apenas para cima) e, quando a fase é `.interacting`, marca usuarioRolou = true, o que desliga a rolagem automática. Ela volta a ser ligada em .onChange(of: conversa.mensagens.count): quando uma nova mensagem é adicionada, usuarioRolou volta a false e rolamos para o fim.

### 1.2. A `Conversa`

É em `Conversa` que de fato ocorre a geração. Veja `Conversa.swift`:

```swift
import Foundation
import FoundationModels
import ClaudeForFoundationModels

@Observable
@MainActor
final class Conversa {
    private let sessao = LanguageModelSession(
        model: ClaudeLanguageModel(
            name: .haiku4_5,
            auth: .apiKey(ProcessInfo.processInfo.environment["ANTHROPIC_API_KEY"] ?? "")
        )
    )
    
    private(set) var respondendo = false
    private(set) var mensagens: [Mensagem] = []
    private var tarefa: Task<Void, Never>?
    
    func enviar(_ texto: String) {
        guard !respondendo else { return }
        respondendo = true
        tarefa = Task {
            defer { respondendo = false }
            mensagens.append(Mensagem(autor: .usuario, texto: texto))
            do {
                let id = UUID()
                mensagens.append(Mensagem(id: id, autor: .assistente, texto: ""))

                for try await parcial in sessao.streamResponse(to: texto) {
                    if let i = mensagens.firstIndex(where: { $0.id == id }) {
                        mensagens[i] = Mensagem(id: id, autor: .assistente, texto: parcial.content)
                    }
                }
            } catch is CancellationError {
                // usuário parou: mantém o texto parcial
            } catch {
                mensagens.append(Mensagem(autor: .assistente, texto: "erro: \(error)"))
            }
        }
        
    }
    
    func parar() {
        tarefa?.cancel()
    }
}
```

A geração ocorre utilizando o framework Foundation Models. Primeiro adiciona-se a mensagem do usuário a `mensagens` e, logo depois, uma mensagem do assistente com string vazia, com um `id` guardado para ser encontrada depois. Em seguida, no loop que percorre o stream da sessão, cada snapshot parcial sobrescreve essa mensagem do assistente (localizada pelo `id`). Dizemos sobrescrevem porque não são deltas acrescentados. A cada atualização o framework retorna o texto completo gerado até aquele momento, então ocorrem séries de substituições.

`sessao` contém um `LanguageModelSession` em que o modelo utilizado é o ClaudeLanguageModel que vem do pacote Swift `ClaudeForFoundationModels` que pode ser obtido de `https://github.com/anthropics/ClaudeForFoundationModels.git`. Alternativamente, pode-se utilizar o modelo que roda no próprio dispositivo (o `SystemLanguageModel` da Apple Intelligence). Para usar esse modelo padrão, cria-se um `LanguageModelSession` sem passar nenhum argumento. Note que é a sessão que guarda o histórico da conversa enviado ao modelo; `mensagens` existe apenas para a interface.

enviar() é bloqueado com guard caso uma geração esteja em andamento e desbloqueado no final com `defer { respondendo = false }`, que roda quando a Task termina, seja por conclusão, erro ou cancelamento via parar(). `respondendo = true` é feito ainda em enviar(), antes de criar a Task, e não dentro dela: a Task só começa a executar depois que enviar() retorna, e nesse intervalo um segundo envio passaria pelo guard.

Quando o usuário chama parar(), o stream lança `CancellationError`. Esse caso é capturado separadamente para não ser exibido como erro; o texto já gerado permanece na mensagem do assistente. 

### 1.3. Bolhas de mensagem com markdown renderizado

As bolhas onde são mostradas as mensagens são desenhadas com a view `LinhaMensagem`, que recebe uma `Mensagem` e a renderiza utilizando blocos de markdown implementados em `BlocoMarkdown.swift`. Veja `LinhaMensagem.swift`.

```swift
import SwiftUI

struct LinhaMensagem: View {
    let mensagem: Mensagem
    
    private var atribuido: AttributedString {
        return (try? AttributedString(markdown: mensagem.texto)) ?? AttributedString(mensagem.texto)
    }
    
    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            ForEach(BlocoMarkdown.blocos(de: atribuido)) { bloco in
                switch bloco.estilo {
                case .paragrafo:
                    Text(bloco.texto)
                case .item(let ordinal):
                    HStack(alignment: .firstTextBaseline, spacing: 6) {
                        Text(ordinal.map { "\($0)." } ?? "•")
                            .frame(width: 24, alignment: .trailing)
                        Text(bloco.texto)
                    }
                }
            }
        }
        .padding(.horizontal, 12)
        .padding(.vertical, 8)
        .background(fundo, in: .rect(cornerRadius: 12))
        .frame(maxWidth: .infinity, alignment: alinhamento)
    }
    
    private var fundo: some ShapeStyle {
        mensagem.autor == .usuario ? AnyShapeStyle(.tint) : AnyShapeStyle(.quaternary)
    }
    
    private var alinhamento: Alignment {
        mensagem.autor == .usuario ? .trailing : .leading
    }
}
```

Cada linha de mensagem, a cada atualização do stream, converte o texto da mensagem em um `AttributedString` com `AttributedString(markdown:)`. Esse parse transforma a sintaxe markdown em atributos: atributos inline (negrito, itálico, código, links) e atributos estruturais (`presentationIntent`: parágrafos, listas, itens de lista, cabeçalhos, etc.). `Text` renderiza corretamente os atributos inline, mas ignora os estruturais, e por isso as quebras de parágrafo e as listas se perderiam. `BlocoMarkdown.blocos()` resolve isso decompondo esse `AttributedString` em blocos, de acordo com os elementos estruturais que determinam layout (listas ordenadas, não ordenadas, parágrafos, etc.), e anotando cada bloco com um Estilo. Assim, cada `BlocoMarkdown` é delimitado por um elemento estrutural, como um parágrafo. E dentro desse bloco individual temos as faixas (runs) de texto com os atributos inline de formatação preservados, que o `Text` desenha. 

Importante notar que a cada snapshot do stream todo o conteúdo gerado até aquele momento é passado para `BlocoMarkdown.blocos()`, que refaz a mesma decomposição retornando a lista com todos os blocos da mensagem. Um bloco já concluído, no entanto, não é reconstruído no `ForEach(BlocoMarkdown.blocos(de: atribuido))`, pois o seu `id` é composto pelo índice sequencial mais o conteúdo textual do bloco, que não muda mais. Assim ele mantém a mesma identidade e o SwiftUI reaproveita a view existente em vez de destruí-la e criar outra. Já o último bloco, que está sendo gerado, muda de texto a cada snapshot, logo muda de `id` e é recriado a cada atualização.

Veja `BlocoMarkdown.swift`:

```swift
import Foundation

struct BlocoMarkdown: Identifiable   {
    enum Estilo {
        case paragrafo
        case item(ordinal: Int?)
    }
    let indice: Int
    var id: String { "\(indice)|\(String(texto.characters))" }
    let estilo: Estilo
    let texto: AttributedString
}

extension BlocoMarkdown {
    static func estilo(de componentes: [PresentationIntent.IntentType]) -> Estilo {
        var ordinal: Int?
        for componente in componentes {
            if case .listItem(let n) = componente.kind {
                ordinal = n
                break
            }
        }
        
        switch componentes.last?.kind {
        case .unorderedList:
            return .item(ordinal: nil)
        case .orderedList:
            return .item(ordinal: ordinal)
        default:
            return .paragrafo
        }
    }
    
    static func blocos(de atribuido: AttributedString) -> [BlocoMarkdown] {
        var resultado: [BlocoMarkdown] = []
        var identidadeAtual: [Int]?
        var acumulado = AttributedString()
        var estiloAtual = Estilo.paragrafo
        
        func fechar() {
            guard identidadeAtual != nil else { return }
            resultado.append(BlocoMarkdown(indice: resultado.count, estilo: estiloAtual, texto: acumulado))
        }
        
        for faixa in atribuido.runs {
            let componentes = faixa.presentationIntent?.components ?? []
            let identidade = componentes.map(\.identity)
            
            if identidade != identidadeAtual {
                fechar()
                identidadeAtual = identidade
                acumulado = AttributedString()
                estiloAtual = Self.estilo(de: componentes)
            }
            
            acumulado += AttributedString(atribuido[faixa.range])
        }
        
        fechar()
        return resultado
    }
}
``` 

Em `BlocoMarkdown.blocos()` percorremos as faixas (runs) do `AttributedString`. Uma faixa é um trecho contínuo de texto em que todos os atributos são iguais. De cada faixa lemos apenas os componentes do seu presentation intent, que são aqueles estruturais que determinam o layout, como itens de lista ordenada/não ordenada, parágrafos, etc. Esses componentes formam uma pilha que vai do elemento mais interno para o mais externo: um item de lista ordenada, por exemplo, vem como `[paragraph, listItem(ordinal: 1), orderedList]`. Pegamos o último, o contêiner mais externo, pois ele determina se o estilo será de um item de lista ordenada/não ordenada ou um parágrafo (qualquer outro contêiner cai no `default` e vira parágrafo). O número do item vem do `listItem` encontrado na pilha.

A identidade do bloco é dada somente pelos elementos de layout como parágrafos, item de lista ordenada/não ordenada, etc., ou seja, pela sequência de `identity` dos componentes do presentation intent. Por isso agrupamos faixas consecutivas que têm a mesma identidade estrutural em um mesmo bloco. Por exemplo, considere o caso em que temos `**ola** tudo bem com *voce*`. Nesse caso três faixas são produzidas, todas com `paragraph (id 1)`, ou seja, o mesmo parágrafo, que é a identidade do bloco. A faixa quebra quando um atributo inline muda, mas como no nosso caso não estamos agrupando por atributo inline, as três faixas de `**ola** tudo bem com *voce*` ficam sob o mesmo bloco, identificado pelo parágrafo de id 1, e cada uma mantém sua formatação inline dentro do texto acumulado. Por isso, quando o loop atinge identidade != identidadeAtual, fechamos o bloco atual e começamos um novo. Isso tudo ocorre a cada snapshot, centenas de vezes no decorrer da geração no stream.

### 1.4. Mensagem

Por fim, temos o nosso modelo de dados `Mensagem`, que representa uma mensagem cujo autor pode ser o assistente ou o usuário:

```swift
import Foundation

enum Autor {
    case usuario
    case assistente
}

struct Mensagem: Identifiable {
    let id: UUID
    let autor: Autor
    let texto: String

    init(id: UUID = UUID(), autor: Autor, texto: String) {
        self.id = id
        self.autor = autor
        self.texto = texto
    }
}
```

### 2. Considerações finais

Neste documento eu descrevo o meu entendimento sobre esse modelo. Interfaces de chat construídas seguindo essa estrutura têm uma propriedade muito boa: o entendimento sobre seu mecanismo fundamental de funcionamento é totalmente compreensível pelo autor dessa skill. Portanto, essa skill, mais do que possibilitar replicar uma estrutura de desenvolvimento, documenta o meu conhecimento sobre construção de interfaces de chat em SwiftUI, e isso é um conhecimento vivo que o agente pode utilizar para dar forma a um app real.

### 3. Nota de melhoria: religar a rolagem automática ao voltar para o fim

No mecanismo atual, depois que o usuário interage com a rolagem, a rolagem automática fica desligada até que uma nova mensagem seja adicionada, mesmo que ele role de volta até o fim durante o stream. Uma melhoria possível é trocar a regra de "o usuário mexeu, então desliga" para "está no fim, então gruda":

- substituir `.onScrollPhaseChange` por `.onScrollGeometryChange`, calculando se a rolagem está perto da borda inferior (por exemplo, `contentOffset.y + containerSize.height >= contentSize.height - margem`);
- `usuarioRolou` passa a significar "não está no fim", atualizado a partir dessa geometria;
- `ScrollPosition`, `scrollTo(edge: .bottom)` e os dois `onChange` permanecem como estão.

Isso muda apenas a regra que decide se a rolagem fica grudada; o restante da estrutura (ContentView, Conversa, LinhaMensagem, BlocoMarkdown, Mensagem) não é afetado.

