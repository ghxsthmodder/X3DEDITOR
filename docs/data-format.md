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
| version | uint16 | Versão do formato; valor inicial definido como `1` |
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

## Metadados do projeto (META)

A seção `META` armazenará informações básicas de identificação e versão do projeto.

Representação-base em C++:

```cpp
struct Metadata {
    std::string name;
    std::string description;

    uint32_t versionMajor;
    uint32_t versionMinor;
    uint32_t versionPatch;
};
```

Convenções:

- `name`: nome do projeto.
- `description`: descrição opcional do projeto.
- `versionMajor`, `versionMinor` e `versionPatch`: versão do projeto.
- Não haverá campo de autor.
- A versão do projeto é independente da versão do formato binário armazenada no cabeçalho do arquivo.

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

Os campos de transformação representam a transformação local do nó. A rotação utiliza quaternion na ordem **XYZW** (`x`, `y`, `z`, `w`), tanto na representação em memória quanto na serialização binária.

### Layout binário do payload NODE

O payload da seção `NODE` começa com `nodeCount` (`uint32`). Em seguida são gravados os registros, na ordem da coleção de nós.

Cada registro contém, nesta ordem:

| Ordem | Campo | Tipo |
|---:|---|---|
| 1 | `name` | String UTF-8 prefixada por `uint32` de tamanho |
| 2 | `parentIndex` | `int32` |
| 3 | `meshCount` | `uint32` |
| 4 | `meshIndices` | `meshCount × uint32` |
| 5 | `position` | 3 × `float32` |
| 6 | `rotation` | 4 × `float32`, ordem XYZW |
| 7 | `scale` | 3 × `float32` |

Todos os campos numéricos seguem a convenção little-endian definida para o arquivo. O tamanho de cada registro é variável devido às strings e à quantidade de índices de malhas.

O leitor deverá validar as contagens e os limites do payload antes de alocar ou acessar os arrays.

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
    std::string name;

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

## Layouts binários definitivos da versão 1

As regras abaixo complementam as representações C++ anteriores e definem a ordem exata dos campos gravados. Não há padding entre campos.

### META

O payload contém um único registro, sem contador:
1. `name`: String UTF-8.
2. `description`: String UTF-8.
3. `versionMajor`: `uint32`.
4. `versionMinor`: `uint32`.
5. `versionPatch`: `uint32`.

Não existe campo de autor. A versão do projeto é independente da versão do formato no cabeçalho.

### NODE

O payload começa com `nodeCount` (`uint32`). Cada nó é gravado na ordem da coleção:
1. `name`: String UTF-8.
2. `parentIndex`: `int32`; `-1` significa raiz.
3. `meshCount`: `uint32`.
4. `meshIndices`: `meshCount` valores `uint32`.
5. `position`: 3 × `float32` (XYZ).
6. `rotation`: 4 × `float32` (XYZW).
7. `scale`: 3 × `float32` (XYZ).

Transformações são locais. O leitor valida índices e rejeita ciclos na hierarquia.

### MESH

O payload começa com `meshCount` (`uint32`). Cada registro é gravado nesta ordem:
1. `name`: String UTF-8.
2. `vertexCount`: `uint32`.
3. `indexCount`: `uint32`.
4. `materialIndex`: `int32`; `-1` significa sem material.
5. `attributeFlags`: `uint32`.
6. `positions`: `vertexCount × 3` valores `float32`, obrigatórios.
7. `normals`: `vertexCount × 3` valores `float32`, somente se o bit 0 estiver ativo.
8. `uvs`: `vertexCount × 2` valores `float32`, somente se o bit 1 estiver ativo.
9. `indices`: `indexCount` valores `uint32`.

Flags da versão 1: bit 0 (`0x1`) indica normais; bit 1 (`0x2`) indica UV0. Todos os outros bits devem ser zero. UV0 é o único canal UV da versão 1.

A versão 1 utiliza listas de triângulos; portanto `indexCount` deve ser múltiplo de três e cada índice deve ser menor que `vertexCount`. A geometria importada pelo Assimp deverá ser triangulada. Cada malha possui um único material; submeshes e múltiplos materiais por malha ficam fora da versão 1. Uma malha deve ter ao menos um vértice.

A representação C++ de `Mesh` inclui `name`, além dos arrays e `materialIndex`. Os vetores de atributos são contíguos.

### MATL

O payload começa com `materialCount` (`uint32`). Cada material contém:
1. `name`: String UTF-8.
2. `albedo`: 4 × `float32` (RGBA).
3. `metallic`: `float32`.
4. `roughness`: `float32`.
5. `opacity`: `float32`.
6. `emissive`: 3 × `float32` (RGB).
7. `albedoTexture`: `int32`.
8. `normalTexture`: `int32`.
9. `metallicTexture`: `int32`.
10. `roughnessTexture`: `int32`.
11. `occlusionTexture`: `int32`.
12. `emissiveTexture`: `int32`.

Os fatores metallic, roughness e opacity devem estar entre 0 e 1. Valores padrão para material novo: albedo (1,1,1,1), metallic 0, roughness 1, opacity 1 e emissive (0,0,0). Cada índice de textura referencia `TEXR`; `-1` significa ausência. As referências metallic e roughness são independentes.

### TEXR

O payload começa com `textureCount` (`uint32`). Cada textura contém:
1. `name`: String UTF-8.
2. `path`: String UTF-8 relativa à pasta do projeto.

Caminhos absolutos não são aceitos. Os bytes das imagens não são incorporados ao `.bin`. O caminho normalizado deve permanecer dentro da pasta do projeto; a validação também deve impedir escapes por segmentos `..` ou links simbólicos.

### Regras comuns de validação

- Coleções são indexadas a partir de zero, na ordem dos registros.
- Cada seção conhecida aparece no máximo uma vez; seções conhecidas ausentes ou duplicadas devem ser rejeitadas.
- Coleções vazias usam contagem zero; META sempre contém seu registro.
- O leitor deve respeitar o limite de cada payload e terminar exatamente no seu fim.
- Validar contagens e operações de tamanho antes de alocar memória, evitando overflow.
- Validar todas as referências: `parentIndex`, `meshIndices`, `materialIndex` e índices de textura.
- Rejeitar floats NaN ou infinitos.
- Seções desconhecidas podem ser ignoradas usando o tamanho declarado.
- Na versão 1 não há compressão, criptografia, checksums, offsets globais, múltiplos canais UV, submeshes ou múltiplos materiais por malha.

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

- [x] Fixar o valor inicial de `version` como `1`.
- [x] Definir os layouts binários de `META`, `NODE`, `MESH`, `MATL` e `TEXR`.
- [x] Definir referências entre nodes, meshes e materiais por índices (`parentIndex`, `meshIndices` e `materialIndex`).
- [ ] Definir suporte a múltiplos canais UV e submeshes.
- [ ] Definir a estratégia de gravação segura (arquivo temporário e substituição).
- [ ] Criar exemplos binários e testes de leitura/escrita.
