# Formato binário interno (.bin)

## Objetivo e escopo

O X3D Editor utiliza um formato binário próprio para armazenar projetos. A versão inicial é **1** e a extensão é `.bin`. O formato prioriza leitura e escrita simples, baixo custo de memória e validação rigorosa.

A versão 1 não inclui compressão, criptografia, checksums, offsets globais, serialização genérica, múltiplos canais UV, submeshes ou múltiplos materiais por malha.

## Convenções gerais

- Inteiros e floats são gravados em little-endian.
- Floats usam IEEE 754 de 32 bits (`float32`).
- Não existe padding ou alinhamento entre campos.
- Strings são UTF-8 válido, precedidas pelo tamanho em bytes (`uint32`), sem terminador NUL.
- Arrays são gravados sequencialmente, sem cabeçalho adicional além das contagens definidas em cada registro.
- Coleções são indexadas a partir de zero, na ordem em que os registros aparecem.
- Contagens e tamanhos devem ser validados antes de qualquer alocação ou acesso.
- Valores `float32` NaN e infinitos são inválidos.

Tipos utilizados: `uint16`, `uint32`, `int32` e `float32`, todos com exatamente os tamanhos indicados pelo nome.

## Cabeçalho

| Campo | Tipo | Descrição |
|---|---|---|
| `magic` | 4 bytes | Bytes ASCII `X3D` seguidos de NUL |
| `version` | `uint16` | Versão do formato; valor inicial `1` |
| `sectionCount` | `uint16` | Número total de seções presentes |

A versão do formato é independente da versão do projeto armazenada em `META`. Um leitor da versão 1 rejeita versões de formato que não suporta.

## Seções

Cada seção tem este cabeçalho:

| Campo | Tipo | Descrição |
|---|---|---|
| `tag` | 4 bytes | Identificador ASCII |
| `size` | `uint32` | Tamanho do payload em bytes |
| `payload` | `size` bytes | Dados da seção |

As cinco seções conhecidas da versão 1 são `META`, `NODE`, `MESH`, `MATL` e `TEXR`. Cada uma deve aparecer exatamente uma vez. Portanto, um arquivo de versão 1 válido contém pelo menos essas cinco seções. Seções adicionais desconhecidas podem ser ignoradas usando `size`; elas contam em `sectionCount`.

Seções conhecidas ausentes ou duplicadas, payloads truncados, tags inválidas ou dados que ultrapassem os limites do arquivo causam rejeição do arquivo. O leitor deve consumir exatamente `size` bytes de cada payload.

## META — metadados do projeto

A seção contém um único registro, sem contador adicional.

Representação C++:

```cpp
struct Metadata {
    std::string name;
    std::string description;

    uint32_t versionMajor;
    uint32_t versionMinor;
    uint32_t versionPatch;
};
```

Payload, nesta ordem:

| Ordem | Campo | Tipo |
|---:|---|---|
| 1 | `name` | String |
| 2 | `description` | String |
| 3 | `versionMajor` | `uint32` |
| 4 | `versionMinor` | `uint32` |
| 5 | `versionPatch` | `uint32` |

Não existe campo de autor. A versão major.minor.patch pertence ao projeto, não ao formato binário.

## NODE — nós e hierarquia

O payload começa com `nodeCount` (`uint32`), seguido pelos nós na ordem da coleção.

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

Cada registro:

| Ordem | Campo | Tipo |
|---:|---|---|
| 1 | `name` | String |
| 2 | `parentIndex` | `int32` |
| 3 | `meshCount` | `uint32` |
| 4 | `meshIndices` | `meshCount × uint32` |
| 5 | `position` | 3 × `float32` (XYZ) |
| 6 | `rotation` | 4 × `float32` (quaternion XYZW) |
| 7 | `scale` | 3 × `float32` (XYZ) |

`parentIndex = -1` indica raiz; valores não negativos referenciam a coleção de nós. `meshIndices` referencia a coleção de malhas. Transformações são locais, relativas ao pai. Um nó pode conter zero ou várias malhas.

O leitor rejeita referências inválidas, auto-referências e ciclos na hierarquia. O quaternion deve ter componentes finitos e comprimento diferente de zero.

## MESH — geometria

O payload começa com `meshCount` (`uint32`).

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

Cada registro:

| Ordem | Campo | Tipo |
|---:|---|---|
| 1 | `name` | String |
| 2 | `vertexCount` | `uint32` |
| 3 | `indexCount` | `uint32` |
| 4 | `materialIndex` | `int32` |
| 5 | `attributeFlags` | `uint32` |
| 6 | `positions` | `vertexCount × 3 × float32` |
| 7 | `normals` | `vertexCount × 3 × float32`, se bit 0 ativo |
| 8 | `uvs` | `vertexCount × 2 × float32`, se bit 1 ativo |
| 9 | `indices` | `indexCount × uint32` |

Flags da versão 1: bit 0 (`0x1`) indica normais; bit 1 (`0x2`) indica UV0. Todos os demais bits devem ser zero. Posições são obrigatórias; normais e UV0 são opcionais. A versão 1 possui apenas um canal UV.

A geometria utiliza triângulos. `indexCount` deve ser múltiplo de três; cada índice deve ser menor que `vertexCount`. `vertexCount` deve ser maior que zero. `indexCount = 0` é permitido para uma malha sem faces. A geometria importada pelo Assimp deve ser triangulada antes de ser armazenada.

`materialIndex = -1` significa ausência de material; caso contrário, referencia `MATL`. Cada malha possui no máximo um material. Submeshes e múltiplos materiais por malha não fazem parte da versão 1.

Os arrays são contíguos. A saída de xatlas pode alterar a correspondência de vértices e deve ser aplicada de forma consistente a posições, índices e UVs.

## MATL — materiais

O payload começa com `materialCount` (`uint32`).

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

Cada registro:

| Ordem | Campo | Tipo |
|---:|---|---|
| 1 | `name` | String |
| 2 | `albedo` | 4 × `float32` (RGBA) |
| 3 | `metallic` | `float32` |
| 4 | `roughness` | `float32` |
| 5 | `opacity` | `float32` |
| 6 | `emissive` | 3 × `float32` (RGB) |
| 7–12 | Referências de textura | 6 × `int32`, na ordem dos campos da estrutura |

Os fatores metallic, roughness e opacity devem estar entre 0 e 1. Valores padrão de um material novo: albedo (1,1,1,1), metallic 0, roughness 1, opacity 1 e emissive (0,0,0). Cada índice de textura referencia `TEXR`; `-1` significa ausência. Metallic e roughness usam referências independentes. Fatores e texturas permanecem separados; sua combinação é responsabilidade do sistema de materiais/renderização.

## TEXR — referências de textura

O payload começa com `textureCount` (`uint32`).

```cpp
struct Texture {
    std::string name;
    std::string path;
};
```

Cada registro contém `name` (String) e `path` (String), nesta ordem. O caminho é relativo à pasta do projeto, em UTF-8. Caminhos absolutos não são aceitos. Os pixels permanecem em arquivos externos e não são incorporados ao `.bin`.

O caminho deve ser normalizado e validado para impedir que a resolução saia da pasta do projeto, inclusive por segmentos `..` ou links simbólicos.

## Regras comuns de validação

- As cinco seções conhecidas são obrigatórias e aparecem exatamente uma vez.
- Seções desconhecidas podem ser ignoradas com segurança usando o tamanho declarado.
- Coleções são indexadas a partir de zero, na ordem dos registros.
- Coleções sem registros usam contador zero; `META` sempre contém um registro.
- O leitor valida tamanhos, contagens e operações aritméticas antes de alocar memória.
- O leitor valida `parentIndex`, `meshIndices`, `materialIndex` e todas as referências de textura.
- Todos os floats devem ser finitos.
- O leitor deve terminar exatamente no fim de cada payload e no fim do arquivo.
- Inteiros com sinal são interpretados em complemento de dois.
- A versão 1 não contém compressão, criptografia, checksums, offsets globais ou padding.

## Gravação segura

A gravação deve preservar o arquivo válido anterior até que o novo arquivo esteja completamente escrito.

1. Criar um arquivo temporário exclusivo no mesmo diretório do arquivo de projeto, usando um nome temporário único.
2. Serializar o arquivo inteiro no temporário; verificar erros de escrita e garantir que todos os bytes foram enviados.
3. Executar flush e sincronizar os dados do arquivo temporário em armazenamento persistente (`fsync` no Android/POSIX).
4. Fechar o temporário. Se qualquer etapa falhar, remover o temporário quando possível e manter o arquivo original intacto.
5. Substituir o destino por meio de rename atômico no mesmo sistema de arquivos.
6. Sincronizar o diretório pai quando suportado, para persistir a alteração de nome.

O temporário deve estar no mesmo diretório para que a substituição possa ser atômica. Nunca truncar o arquivo de destino antes de a gravação temporária terminar. Se o rename falhar, o arquivo anterior permanece e a operação retorna erro. A implementação deve tratar resíduos temporários de execuções interrompidas sem confundi-los com projetos válidos.

## Exemplo de referência e testes

O arquivo `tests/fixtures/v1-minimal.hex` contém uma fixture hexadecimal byte a byte de um projeto mínimo válido da versão 1. Ele pode ser convertido em binário para servir como entrada de teste. Os casos de aceitação, rejeição, round-trip e gravação segura estão descritos em `docs/format-v1-validation.md`.

## Integração com bibliotecas

- **Assimp:** importa formatos externos para a representação interna e exporta a cena para formatos suportados.
- **xatlas:** recebe posições e índices para gerar UVs; sua saída deve ser aplicada de forma consistente à geometria da malha.

O formato `.bin` é interno ao projeto e não substitui os formatos de intercâmbio atendidos pelo Assimp.

## Estado da especificação

- [x] Revisão técnica da especificação da versão 1 e consolidação das regras.
- [x] Layouts binários de `META`, `NODE`, `MESH`, `MATL` e `TEXR`.
- [x] Exemplo binário mínimo em fixture hexadecimal.
- [x] Matriz de testes de leitura, escrita, validação e gravação segura documentada.
- [x] Estratégia de gravação segura por temporário, sincronização e rename atômico.
- [ ] Implementar o leitor, gravador e testes automatizados em C++ junto ao backend.
