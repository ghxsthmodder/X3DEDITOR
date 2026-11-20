# Formato binário interno (.bin)

## Objetivo

O X3D Editor utilizará um formato binário próprio desde o início. O objetivo é manter o armazenamento simples, compacto e rápido de ler/escrever, sem criar um contêiner excessivamente complexo.

A extensão dos arquivos de projeto será **`.bin`**.

## Princípios

- Cabeçalho pequeno com identificação e versão.
- Dados organizados em seções identificadas.
- Campos numéricos com tipos e tamanhos definidos.
- Arrays de vértices, índices e UVs armazenados diretamente em binário.
- Sem compressão, criptografia, checksums ou tabelas complexas na primeira versão.
- Evolução do formato por número de versão.

## Estrutura geral do arquivo

O arquivo será composto por um cabeçalho seguido pelas seções:

| Campo | Tipo | Descrição |
|---|---|---|
| magic | 4 bytes | Identificação ASCII `X3D\0` |
| version | uint16 | Versão do formato |
| sectionCount | uint16 | Quantidade de seções |
| sections | variável | Seções sequenciais |

Todos os valores numéricos serão little-endian. Floats serão IEEE 754 de 32 bits.

### Estrutura de cada seção

| Campo | Tipo | Descrição |
|---|---|---|
| tag | 4 bytes | Identificador ASCII da seção |
| size | uint32 | Tamanho do payload em bytes |
| payload | `size` bytes | Conteúdo da seção |

O leitor deverá validar o cabeçalho, a versão, a quantidade de seções e os limites de cada payload antes de acessar os dados.

## Seções iniciais

As tags abaixo são a proposta inicial do schema:

| Tag | Conteúdo |
|---|---|
| `META` | Nome do projeto e metadados básicos |
| `NODE` | Objetos da cena, hierarquia e transformações |
| `MESH` | Malhas, atributos de vértices, UVs e índices |
| `MATL` | Materiais e seus parâmetros |
| `TEXR` | Referências a arquivos de textura |

Uma seção poderá conter uma contagem de registros (`uint32`) seguida pelos registros correspondentes. Strings serão UTF-8 prefixadas pelo tamanho (`uint32`), sem terminador nulo.

## Hierarquia de nós

A hierarquia da cena utilizará **índices**, não ponteiros entre objetos. Essa decisão mantém a representação independente de endereços de memória e facilita a serialização no formato binário.

Representação-base em C++:

```cpp
struct Node {
    std::string name;
    int32_t parentIndex;
    std::vector<uint32_t> meshIndices;

    float position[3];
    float rotation[4];
    float scale[3];
};
```

Convenções definidas:

- `parentIndex`: índice do nó pai na coleção de nós; `-1` indica que o nó não possui pai (nó raiz).
- `meshIndices`: índices das malhas associadas ao nó na coleção de malhas.
- Os índices são referências lógicas às coleções, não offsets de bytes no arquivo.
- O leitor deverá validar se os índices referenciados existem e se a hierarquia não contém referências inválidas.

Os campos de transformação representam a transformação local do nó. A ordem e a convenção exata dos componentes do quaternion serão especificadas junto ao layout binário definitivo de `NODE`.

## Dados de malha

A seção `MESH` deverá armazenar, no mínimo, os dados necessários para reconstruir uma malha:
- nome ou identificador;
- quantidade de vértices;
- posições (3 × float32 por vértice);
- normais, quando disponíveis;
- coordenadas UV por canal, quando disponíveis;
- quantidade de índices e índices (uint32);
- referência ao material, quando aplicável.

O layout exato dos registros de malha será fechado antes da implementação do leitor/escritor. A organização deverá permitir que xatlas receba posições e índices e que os UVs gerados sejam gravados de volta na malha.

### Representação interna em C++

A representação de malhas no backend utilizará arrays contíguos com `std::vector`, evitando alocações por vértice e facilitando o envio dos dados ao xatlas e ao serializador binário.

Representação-base aprovada:

```cpp
struct Mesh {
    std::vector<float> positions;
    std::vector<float> normals;
    std::vector<float> uvs;
    std::vector<uint32_t> indices;

    int32_t materialIndex;
};
```

Convenções:
- `positions`: sequência de componentes XYZ; tamanho igual a `vertexCount * 3`.
- `normals`: sequência de componentes XYZ; quando presente, tamanho igual a `vertexCount * 3`.
- `uvs`: sequência de componentes UV; no canal inicial, tamanho igual a `vertexCount * 2`.
- `indices`: índices de vértices usando `uint32_t`; a contagem deve ser compatível com a topologia suportada.
- `materialIndex`: índice do material na coleção de materiais; `-1` significa que a malha não possui material atribuído.

Os vetores armazenam dados em ordem contígua. O payload binário gravará os componentes diretamente, em little-endian, com contagens explícitas para que o leitor possa validar os tamanhos antes de alocar memória.

Essa é a representação-base inicial. Submeshes, múltiplos canais UV, cores de vértices, tangentes e outros atributos poderão ser adicionados ao schema quando suas necessidades forem definidas, sem presumir que já fazem parte desta estrutura mínima.

A integração com xatlas utilizará os arrays de posições e índices como entrada; as coordenadas UV resultantes serão copiadas/transferidas para `uvs`, respeitando a correspondência de vértices produzida pelo processo de geração.

## Materiais

A representação interna de materiais utilizará parâmetros PBR e referências por índice para cada textura. As texturas metallic e roughness serão referenciadas separadamente.

Representação-base em C++:

```cpp
struct Material {
    std::string name;

    float albedo[4];
    float metallic;
    float roughness;
    float opacity;
    float emissive[3];

    int32_t albedoTexture;
    int32_t normalTexture;
    int32_t metallicTexture;
    int32_t roughnessTexture;
    int32_t occlusionTexture;
    int32_t emissiveTexture;
};
```

Convenções:

- `albedo`: cor base RGBA.
- `metallic`: fator metálico.
- `roughness`: fator de rugosidade.
- `opacity`: fator de opacidade.
- `emissive`: cor de emissão RGB.
- Cada campo `*Texture` referencia um registro da coleção de texturas.
- Índice `-1` indica que não há textura associada àquele parâmetro.
- `metallicTexture` e `roughnessTexture` são referências independentes.

Os valores numéricos dos parâmetros serão armazenados como `float32`. Os limites, valores padrão e regras de combinação entre fator e textura serão especificados no fechamento do payload `MATL`.

## Texturas

Na primeira versão, a seção `TEXR` armazenará referências a arquivos de textura externos ao `.bin`, sem incorporar os bytes das imagens ao arquivo de projeto.

Representação-base em C++:

```cpp
struct Texture {
    std::string name;
    std::string path;
};
```

Convenções:

- `name`: nome identificador da textura.
- `path`: caminho **relativo** à pasta do projeto, armazenado em UTF-8.
- Caminhos absolutos não serão aceitos.
- Os arquivos de imagem deverão estar dentro da estrutura de diretórios do projeto.
- A estrutura não armazenará os pixels da imagem; o carregamento ocorrerá quando a textura for necessária.

O leitor deverá validar o caminho resolvido para impedir que referências relativas escapem da pasta do projeto (por exemplo, por meio de segmentos `..`).

## Integração com bibliotecas

- **Assimp:** importa formatos externos para a representação de cena do X3D Editor e exporta a representação interna para os formatos suportados.
- **xatlas:** recebe os dados de geometria necessários e devolve o mapeamento UV, que será armazenado na seção de malhas.

O formato `.bin` é o formato interno do projeto; não substitui os formatos de intercâmbio atendidos pelo Assimp.

## Escopo deliberadamente limitado

A primeira versão não terá:
- compressão;
- criptografia;
- checksums;
- índices de offsets globais;
- serialização genérica de objetos;
- compatibilidade automática com versões futuras.

Seções desconhecidas poderão ser ignoradas usando o campo `size`, desde que o leitor consiga percorrê-las com segurança. A política para versões incompatíveis será rejeitar o arquivo com uma mensagem clara.

## Pendências antes da implementação

- [ ] Fixar o valor inicial de `version`.
- [ ] Definir os payloads de `META`, `NODE`, `MESH`, `MATL` e `TEXR`.
- [x] Definir referências entre nodes, meshes e materiais por índices (`parentIndex`, `meshIndices` e `materialIndex`).
- [ ] Definir suporte a múltiplos canais UV e submeshes.
- [ ] Definir a estratégia de gravação segura (arquivo temporário e substituição).
- [ ] Criar exemplos binários e testes de leitura/escrita.
