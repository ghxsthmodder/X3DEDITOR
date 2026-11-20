# X3D Editor

**Editor de modelos 3D com suporte a texturização e geração de UVs.**

O X3D Editor é um projeto em fase inicial de definição. A proposta é criar uma ferramenta para trabalhar com modelos 3D, preparar seus mapas UV e aplicar/organizar texturas, priorizando desempenho e um fluxo de trabalho direto.

## Objetivos

- Importar e trabalhar com modelos 3D.
- Gerar mapas UV automaticamente utilizando [xatlas](https://github.com/jpcy/xatlas).
- Visualizar e organizar as ilhas UV.
- Oferecer ferramentas de texturização.
- Manter uma interface objetiva, responsiva e eficiente.

## Interface

A direção inicial da interface foi definida com os seguintes princípios:

- **Orientação principal:** horizontal (*landscape*), para aproveitar melhor a área de trabalho.
- **Design:** Material Design. A versão específica será escolhida conforme a plataforma e a implementação.
- **Prioridade:** desempenho e funcionalidade antes de refinamentos estéticos.
- **Simplicidade:** evitar painéis, animações e elementos que não contribuam diretamente para o trabalho.

### Tela inicial

A primeira tela será um gerenciador de projetos, contendo:

- Lista de projetos existentes.
- Ação para criar um novo projeto.
- Pesquisa por nome.
- Acesso para abrir um projeto.
- Menu contextual para renomear ou excluir projetos.

As miniaturas serão opcionais e deverão ser carregadas sob demanda. A tela inicial não deverá carregar geometrias ou texturas dos modelos apenas para exibir a lista.

## UV Unwrapping

O [xatlas](https://github.com/jpcy/xatlas) foi escolhido para o processo de geração automática de UVs. A integração deverá permitir:

1. Processar a geometria de entrada.
2. Gerar o mapeamento UV (*UV unwrapping*).
3. Criar e organizar as ilhas UV.
4. Disponibilizar as coordenadas UV resultantes para as etapas de visualização e texturização.

Os parâmetros de geração, as opções de empacotamento e o tratamento de malhas e materiais serão especificados durante o desenvolvimento.

## Diretrizes de desempenho

- Executar operações pesadas fora da thread da interface.
- Carregar modelos, texturas e miniaturas somente quando necessário.
- Evitar cópias desnecessárias de dados de geometria e imagem.
- Manter o uso de memória previsível.
- Preferir componentes de interface simples e eficientes.
- Evitar efeitos visuais e animações sem benefício funcional.

## Escopo atual

O projeto está na fase de planejamento. A definição da tela inicial é o primeiro passo; a organização interna do editor 3D será documentada separadamente.

Ainda estão em aberto:

- Plataforma e tecnologias de implementação.
- Formatos de importação e exportação de modelos.
- Formato e estrutura dos arquivos de projeto.
- Arquitetura do viewport 3D.
- Ferramentas de edição e inspeção de malhas.
- Fluxo de texturização e formatos de imagem.
- Interface de visualização e edição de UVs.

## Roadmap inicial

- [x] Definir o nome **X3D Editor**.
- [x] Definir a orientação horizontal como padrão.
- [x] Escolher Material Design como direção visual.
- [x] Definir a lista de projetos como tela inicial.
- [x] Escolher xatlas para geração de UVs e ilhas.
- [ ] Especificar o fluxo de criação e gerenciamento de projetos.
- [ ] Definir a estrutura de diretórios e metadados dos projetos.
- [ ] Especificar a área de edição e o viewport 3D.
- [ ] Planejar a integração técnica do xatlas.
- [ ] Especificar as ferramentas de texturização.

## Status

**Em planejamento — definições iniciais em andamento.**

As decisões e especificações serão mantidas neste repositório à medida que o projeto evoluir.
