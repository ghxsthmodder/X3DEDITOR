# Testes de conformidade — formato X3D v1

Este documento define os testes de conformidade para o leitor e gravador do formato binário. A fixture de referência é `tests/fixtures/v1-minimal.hex`.

## Fixture mínima

A fixture contém 169 bytes e representa:

- Cabeçalho `X3D\0`, versão de formato 1 e cinco seções.
- `META`: nome `Example`, descrição `Minimal X3D v1 fixture`, versão do projeto 1.0.0.
- `NODE`: um nó chamado `Root`, sem pai e sem malhas; posição (0,0,0), quaternion identidade (0,0,0,1) e escala (1,1,1).
- `MESH`, `MATL` e `TEXR`: coleções vazias.

A fixture é uma sequência hexadecimal sem endereços ou separadores semânticos. Espaços e quebras de linha são ignorados ao convertê-la em bytes.

## Testes positivos obrigatórios

| ID | Cenário | Resultado esperado |
|---|---|---|
| P-01 | Ler a fixture mínima | Aceitar e reconstruir exatamente os metadados e o nó descritos acima |
| P-02 | Ler coleções vazias | Aceitar MESH, MATL e TEXR com contador zero |
| P-03 | Round-trip da fixture | Ler e gravar novamente; produzir os mesmos 169 bytes |
| P-04 | Seção desconhecida adicional, com tamanho válido | Ignorar o payload desconhecido e continuar a leitura |
| P-05 | Nó com pai válido e referências válidas | Reconstruir a hierarquia e as associações corretamente |
| P-06 | Malha com normais e UV0 habilitados | Ler os arrays de acordo com os bits 0 e 1 de attributeFlags |
| P-07 | Material com referências de textura válidas | Resolver cada índice para o registro correto de TEXR |

## Testes negativos obrigatórios

| ID | Cenário inválido | Resultado esperado |
|---|---|---|
| N-01 | Magic diferente de `X3D\0` | Rejeitar |
| N-02 | Versão de formato não suportada | Rejeitar com erro de versão |
| N-03 | Cabeçalho truncado | Rejeitar |
| N-04 | sectionCount incompatível com as seções disponíveis | Rejeitar |
| N-05 | Uma das cinco seções obrigatórias ausente | Rejeitar |
| N-06 | Seção conhecida duplicada | Rejeitar |
| N-07 | size maior que os bytes restantes | Rejeitar sem leitura fora dos limites |
| N-08 | Payload conhecido com bytes sobrando ou faltando | Rejeitar |
| N-09 | String cujo tamanho ultrapassa o payload | Rejeitar |
| N-10 | String UTF-8 inválida | Rejeitar |
| N-11 | Contagem que causa overflow ou alocação além dos limites disponíveis | Rejeitar antes de alocar |
| N-12 | parentIndex, meshIndices, materialIndex ou índice de textura fora do intervalo | Rejeitar |
| N-13 | Hierarquia com ciclo ou nó como próprio pai | Rejeitar |
| N-14 | attributeFlags com bits desconhecidos | Rejeitar |
| N-15 | indexCount não múltiplo de três ou índice maior/igual a vertexCount | Rejeitar |
| N-16 | metallic, roughness ou opacity fora de [0,1] | Rejeitar |
| N-17 | Float NaN ou infinito em qualquer atributo | Rejeitar |
| N-18 | Caminho absoluto ou caminho de textura que escape da pasta do projeto | Rejeitar |
| N-19 | Bytes extras após a última seção | Rejeitar |

## Testes da gravação segura

| ID | Simulação | Resultado esperado |
|---|---|---|
| S-01 | Gravação concluída e rename bem-sucedido | Novo arquivo substitui o anterior integralmente |
| S-02 | Falha ao criar o temporário | Arquivo anterior permanece intacto |
| S-03 | Falha durante a escrita ou flush | Temporário é descartado quando possível; destino anterior permanece intacto |
| S-04 | Falha em fsync antes do rename | Não substituir o destino |
| S-05 | Falha no rename | Retornar erro e preservar o destino anterior |
| S-06 | Processo interrompido antes do rename | Arquivo de destino anterior continua válido; temporário órfão pode ser limpo posteriormente |
| S-07 | Gravação em diretório sem permissão | Retornar erro sem truncar o arquivo anterior |

## Critérios de aprovação

- Todos os testes positivos são aceitos e todos os negativos são rejeitados.
- O round-trip P-03 deve ser byte a byte idêntico para a fixture canônica.
- Erros de leitura nunca podem resultar em acesso fora dos limites ou alocação não limitada.
- Em falhas de gravação anteriores ao rename, o arquivo anterior deve permanecer intacto.
- Os testes automatizados do leitor e gravador devem ser implementados em C++ junto ao backend. A fixture hexadecimal serve como vetor de referência independente da implementação.

