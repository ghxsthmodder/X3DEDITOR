# Formato de dados interno

## Decisão inicial

O X3D Editor utilizará **JSON como formato de dados interno inicial**.

A decisão prioriza simplicidade de inspeção, depuração e evolução durante as primeiras etapas do projeto. Futuramente, o projeto poderá adotar um formato binário para reduzir tamanho, tempo de leitura/escrita e custo de processamento.

## Princípios

- Os dados devem possuir uma estrutura documentada e versionada.
- Cada documento persistido deve identificar sua versão de formato.
- A representação deve distinguir cena, objetos, malhas, materiais, texturas e dados UV.
- Referências entre objetos devem ser explícitas e estáveis.
- A interface não deve depender diretamente de detalhes de serialização.
- A evolução para binário deve prever conversão ou migração dos projetos JSON existentes.

## Estrutura conceitual

A estrutura lógica poderá conter:

- **Project:** informações gerais e configurações do projeto.
- **Scene:** conjunto de objetos e hierarquia.
- **Node/Object:** transformações e referências aos recursos.
- **Mesh:** atributos de vértices, índices e submeshes.
- **Material:** parâmetros e referências a texturas.
- **Texture:** caminho, formato, dimensões e demais metadados.
- **UV:** canais de coordenadas e referências às ilhas/atlas gerados.

Esta é uma divisão conceitual, não um schema definitivo.

## JSON e dados de grande volume

JSON é adequado para metadados e estruturas descritivas, mas arrays extensos de vértices, índices e pixels podem gerar arquivos grandes e maior custo de parsing e memória.

Na primeira versão, a representação pode permanecer em JSON para manter o formato simples. Ao definir o schema, deve-se evitar acoplar a lógica do editor à forma textual de armazenamento. Se necessário, dados binários poderão ser introduzidos em uma evolução futura, mantendo JSON como manifesto/descritor do projeto.

## Evolução binária futura

Uma possível evolução é um contêiner binário versionado que armazene blocos de dados de malhas, UVs e outros recursos, mantendo metadados ou um manifesto separado.

Essa evolução ainda não está especificada e não deve ser tratada como implementação já decidida.

## Pendências

- [ ] Definir o nome/extensão dos arquivos de projeto.
- [ ] Definir schema JSON e estratégia de versionamento.
- [ ] Definir representação de vetores, matrizes, índices e atributos de vértice.
- [ ] Definir referências e IDs de recursos.
- [ ] Definir como armazenar texturas: arquivos externos ou incorporados.
- [ ] Definir política de migração para versões futuras.
- [ ] Avaliar quando JSON deixa de ser suficiente e quando introduzir armazenamento binário.
