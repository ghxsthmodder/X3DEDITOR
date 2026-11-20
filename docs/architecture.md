# Arquitetura tecnológica

## Visão geral

O X3D Editor utilizará uma arquitetura híbrida, separando a camada de interface da camada de processamento 3D.

## Linguagens

### Java / Kotlin — interface

Java e/ou Kotlin serão utilizados na camada de interface. A escolha entre usar uma linguagem, ou combinar ambas, poderá ser refinada durante a implementação.

Responsabilidades previstas:
- Interface Material Design.
- Gerenciamento e listagem de projetos.
- Menus, diálogos, configurações e controles.
- Fluxos de edição e texturização.
- Comunicação com o backend nativo.

### C++ — backend

C++ será utilizado para as operações de processamento 3D e manipulação de dados.

Responsabilidades previstas:
- Processamento de geometrias e malhas.
- Integração com xatlas para UV unwrapping e geração de ilhas.
- Integração com Assimp para importação/exportação de modelos.
- Preparação dos dados de modelos para os fluxos de edição.
- Operações pesadas que não devem bloquear a interface.

## Bibliotecas

### xatlas

Repositório: https://github.com/jpcy/xatlas

Biblioteca escolhida para geração automática de mapas UV:
- UV unwrapping.
- Geração de ilhas UV.
- Empacotamento/organização das ilhas conforme os recursos da biblioteca.

### Assimp

Repositório: https://github.com/assimp/assimp

Biblioteca escolhida para facilitar a interoperabilidade com formatos de modelos 3D:
- Importação e exportação por meio dos formatos e recursos suportados pela versão integrada.
- Conversão entre estruturas de cena e malhas.
- A exportação de cada formato deverá ser validada individualmente, incluindo materiais, texturas, hierarquia e atributos de malha.

## Comunicação entre interface e backend

A interface Java/Kotlin chamará as funções nativas C++ por **JNI (Java Native Interface)**. Como o projeto será exclusivo para Android, não haverá necessidade de abstrair a integração para outras plataformas. O contrato da API JNI ainda será definido.

Princípios:
- Não executar operações pesadas na thread principal da interface.
- Evitar cópias desnecessárias de buffers grandes entre JVM e C++.
- Definir claramente propriedade, duração e liberação de memória.
- Retornar progresso, erros e resultados de forma controlada.
- Manter a API nativa independente dos detalhes visuais da interface sempre que possível.

## Fluxo conceitual de processamento

1. A interface solicita uma operação.
2. A camada de integração encaminha os dados necessários ao backend.
3. O backend C++ carrega ou processa a geometria.
4. O xatlas gera o mapeamento UV quando solicitado.
5. O Assimp é utilizado para importar/exportar formatos suportados.
6. O backend retorna o resultado e eventuais mensagens de erro à interface.

## Formato de dados interno

O formato interno inicial será **JSON**. Ele será utilizado para representar a estrutura dos dados do projeto e do modelo durante a primeira fase.

A arquitetura deverá permitir a evolução futura para um formato binário, caso JSON apresente limitações de desempenho, tamanho ou consumo de memória. Para facilitar essa transição:
- versionar o formato e seus documentos;
- separar a descrição estrutural dos dados de grande volume sempre que possível;
- evitar dependência da interface em detalhes específicos da serialização;
- definir regras de conversão/migração entre versões.

A especificação dos campos e da organização dos dados será detalhada em [data-format.md](data-format.md).

## Decisões ainda pendentes

- Versão mínima do Android.
- Contrato detalhado da API JNI.
- Se Java, Kotlin ou ambos serão usados no código da interface.
- Formatos de entrada e saída prioritários.
- Modelo detalhado de dados de malhas, materiais e cenas.
- Política de preservação de atributos e materiais na exportação.
- Tratamento de cancelamento e progresso de operações longas.
