# X3D Editor 1.0 — Especificação técnica mestre e plano de desenvolvimento

> **Status:** especificação funcional consolidada; implementação ainda não iniciada ou não verificada por este documento.  
> **Plataforma:** Android, orientação horizontal.  
> **Formato nativo:** X3D, extensão `.bin`.  
> **API gráfica:** Vulkan 1.1.  
> **Última revisão:** 2026-11-20.

Este documento reúne as decisões aprovadas para a versão 1.0 e estabelece uma sequência de desenvolvimento. Ele é a referência de escopo do produto; não substitui as especificações detalhadas de cada subsistema nem declara recursos como implementados.

## 1. Objetivo e princípios

O X3D Editor é um editor de modelagem 3D exclusivo para Android. A versão 1.0 deve oferecer um fluxo confiável de criação, edição, importação, exportação e armazenamento de cenas e malhas.

Princípios orientadores:

- Priorizar estabilidade, desempenho e previsibilidade em dispositivos móveis.
- Manter a interface responsiva, sem executar operações pesadas na thread principal.
- Separar interface Android, lógica do editor, dados da cena e renderização.
- Evitar cópias integrais e alocações desnecessárias de geometria.
- Preservar projetos existentes durante salvamentos e falhas.
- Implementar primeiro uma base de modelagem sólida; recursos de texturização e animação ficam fora do escopo 1.0.
- Tratar limitações dos formatos externos explicitamente, sem prometer conversões sem perda quando o formato de destino não comportar os dados.

## 2. Escopo funcional da versão 1.0

### 2.1 Plataforma e interface

- Android exclusivo, orientação horizontal (*landscape*).
- Interface Material Design; a versão específica da biblioteca visual será escolhida na implementação.
- Tema escuro padrão; temas claro e sistema também disponíveis.
- Idiomas: Português (Brasil), Inglês e Espanhol, com troca sem reiniciar.
- Tamanhos de fonte: Pequeno, Médio (padrão) e Grande.
- Armazenamento padrão no espaço privado do aplicativo.
- Importação e exportação de arquivos pelo Android Storage Access Framework (SAF), sem exigir permissão ampla de armazenamento.

### 2.2 Gerenciador de projetos

A tela inicial terá barra lateral fixa e grade de projetos. Cada cartão apresentará miniatura, nome e data/hora da última modificação, além de menu contextual para renomear ou excluir. Abrir será feito ao tocar no cartão.

- Novo projeto abre diálogo com o nome obrigatório.
- Criar abre diretamente a tela de modelagem.
- Excluir exige confirmação e remove os arquivos associados ao projeto.
- Miniaturas são imagens previamente geradas, armazenadas na pasta do projeto e carregadas sob demanda.
- Projetos sem miniatura usam imagem padrão.
- A listagem não deve carregar geometria nem desserializar arquivos X3D apenas para exibir cartões.

Configurações incluem Aparência, Idioma, Armazenamento e Sobre. Armazenamento apresenta quantidade de projetos, espaço ocupado e cache/miniaturas; limpar cache não remove projetos nem arquivos originais. Sobre apresenta versões do aplicativo e formato, bibliotecas, licenças e repositório.

### 2.3 Tela de modelagem

A viewport é a área central. A barra superior contém voltar ao gerenciador, salvar, desfazer, refazer e configurações. A saída com alterações não salvas exige confirmação.

A lateral esquerda possui as categorias Modelagem e Adicionar. Os painéis laterais são recolhíveis; a hierarquia abre sobreposta pela direita. A barra inferior é contextual e contém Posição, Rotação e Escala. Não haverá painel visual de histórico na versão 1.0.

**Adicionar:** primitivas (incluindo cubo, esfera e cilindro), Empty/nó de posição, câmera, luz e importação de modelo.

**Modelagem:** modos exclusivos Objeto (padrão), Vértices, Arestas e Faces. Ferramentas:
- Geometria: Extrude, Inset e Bevel.
- Topologia: Loop Cut, Merge e Subdivide.
- Outras: Delete e Move Pivot.

As ferramentas terão parâmetros contextuais e pré-visualização temporária quando aplicável. Confirmar aplica a operação; cancelar descarta o preview sem modificar a geometria original.

### 2.4 Seleção

- Seleção individual e múltipla por controle específico.
- Seleção retangular: arrastar esquerda→direita seleciona elementos totalmente contidos; direita→esquerda seleciona elementos contidos ou cruzados.
- A seleção por área respeita o modo ativo e pode ser combinada com seleção múltipla.
- Toque em vazio limpa a seleção.
- Hierarquia e viewport permanecem sincronizadas.
- Objetos selecionados usam contorno laranja; foco/hover usa azul; componentes têm destaque próprio.
- O objeto ativo deve ser distinguível dos demais itens da seleção múltipla.

### 2.5 Transformações e unidades

- Manipuladores visuais e campos numéricos sincronizados em tempo real.
- Posição, rotação e escala na barra inferior.
- Coordenadas Global e Local; Global é o padrão.
- Posição em unidades do mundo, rotação exibida em graus e escala por eixo.
- Bloqueio de escala uniforme.
- Snap ativável/desativável, inicialmente com posição 0,1 m, rotação 15° e escala 0,1; incrementos configuráveis.
- Restrição por eixo X/Y/Z, por plano XY/XZ/YZ ou livre.
- Precisão de exibição configurável: 2, 3, 4 ou 6 casas decimais; padrão 3.
- Valores internos de posição, rotação e escala usam float32/quaternion conforme a arquitetura. Alterar unidade muda apenas a apresentação, não a geometria.
- Unidades de exibição: metros, centímetros e milímetros.

Move Pivot é distinto de Empty: altera a origem/pivô sem deslocar visualmente a geometria. Modos: mover livremente, centralizar na geometria e origem do mundo.

### 2.6 Operações de malha

Todas as oito ferramentas estão no escopo 1.0:

| Ferramenta | Comportamento mínimo |
|---|---|
| Extrude | Extrusão de faces/regiões selecionadas; distância e modo de extrusão |
| Inset | Contorno interno; espessura e opção de contorno individual |
| Bevel | Chanfrar arestas/vértices; largura, segmentos e limite por ângulo |
| Loop Cut | Inserir cortes seguindo a topologia; quantidade e posicionamento |
| Merge | Unir vértices; centro, último selecionado ou distância |
| Subdivide | Dividir arestas/faces; número de subdivisões |
| Delete | Remover objetos ou componentes conforme o modo de seleção |
| Move Pivot | Alterar pivô sem deslocamento visual da geometria |

Operações confirmadas integram o Undo/Redo. A implementação deve validar a topologia e impedir referências pendentes ou dados inválidos.

### 2.7 Modificadores

A pilha de modificadores (Modifier Stack) estará disponível na versão 1.0, com **Mirror** como único modificador inicial.

Mirror:
- Eixos X, Y e Z.
- Pré-visualização não destrutiva.
- Atualização quando a geometria original mudar.
- Ativar/desativar e aplicar definitivamente.
- Clipping na linha central e soldagem de vértices próximos ao plano de simetria.
- Ordem editável na pilha, preparada para futuros modificadores.

Bevel Modifier e Subdivision Surface ficam para versões futuras. A ferramenta destrutiva Bevel continua incluída na versão 1.0.

### 2.8 Cena e hierarquia

A cena utiliza Scene Graph hierárquico. Nós possuem identificador único, nome, transformação local, pai, filhos e componentes opcionais.

Componentes previstos: Mesh, Camera, Light e Empty. Empty não contém geometria e pode representar referências como pontos de acoplamento. A transformação global é derivada da hierarquia.

- IDs persistentes de nós: uint64.
- Referências internas usam IDs, nunca nomes.
- Transformação local: posição float32×3, quaternion float32×4 (XYZW) e escala float32×3.
- Matriz global calculada pelo motor quando necessária.
- Transform Dirty Flags propagam invalidação aos descendentes e evitam recomputações desnecessárias.
- Sistema destro (*Right-Handed*), Y vertical, Z negativo frontal.
- UI apresenta ângulos em graus; motor usa radianos quando trabalhar com ângulos e quaternions internamente.

### 2.9 Câmeras e navegação

A câmera de navegação da viewport é independente das câmeras da cena e usa modelo orbital (Orbit Camera), com Target, distância, rotação horizontal/vertical, zoom e pan.

Gestos padrão:
- Um dedo: orbitar.
- Pinça com dois dedos: zoom.
- Dois dedos movendo juntos: pan.

Controles rápidos: enquadrar tudo, enquadrar selecionado e restaurar câmera. Projeções Perspective e Orthographic. Sensibilidade de rotação, zoom e pan configurável; distância mínima/máxima e limite de rotação vertical configuráveis.

Câmeras da cena: múltiplas câmeras, criar, renomear, excluir, definir ativa, visualizar através da câmera, ajustar FOV/perspectiva e planos Near/Far.

### 2.10 Luzes, viewport e apresentação

Luzes de cena: Point, Directional e Spot, com cor, intensidade, transformação e parâmetros específicos. Gizmos permanecem visíveis mesmo com a luz desativada. A iluminação auxiliar da viewport é independente da cena, ativa por padrão em Solid, não é objeto e não é salva no projeto.

Iluminação auxiliar de três pontos: Key, Fill e Rim. Modos de viewport: Solid, Wireframe, Wireframe + Solid e Flat Shading. Fundo: cinza escuro uniforme, gradiente escuro ou grade 3D com horizonte/eixos; grade e eixos podem ser ocultados independentemente.

Recursos gráficos:
- Sombras em tempo real via Shadow Mapping. CSM para Directional; mapas convencionais para Point e Spot.
- SSAO.
- MSAA: desligado, 2x ou 4x; 4x é o padrão desejado, condicionado ao suporte do dispositivo.
- FXAA como opção de pós-processamento.
- Tone Mapping e Gamma Correction.
- Perfis Baixo, Médio, Alto e Personalizado; efeitos configuráveis individualmente.
- Detecção inicial das capacidades Vulkan e recomendação de perfil. Usuário pode alterar o perfil depois.

A qualidade de sombras oferece Desativada, Baixa, Média e Alta; Média é o padrão proposto. A implementação deve respeitar limites e formatos realmente suportados pelo dispositivo, com fallback seguro.

### 2.11 Materiais e texturas

A versão 1.0 inclui apenas materiais básicos para visualização:
- Cor base por objeto/material.
- Material padrão.
- Parâmetros básicos de iluminação.
- Opacidade básica.
- Exibição de cores no modo Solid.

Não inclui editor PBR avançado, texturização ou carregamento/edição de texturas. A estrutura de dados deve ser compatível com evolução futura, sem expor funcionalidades não implementadas.

### 2.12 Importação e exportação

Formatos prioritários: glTF/GLB, FBX, OBJ e DAE (COLLADA), usando Assimp como biblioteca principal.

Fluxo: **modelo externo → importador/normalização → cena X3D → exportador → modelo externo**.

Regras:
- Importar a cena inteira ou elementos selecionados quando viável.
- Preservar hierarquia, transformações e geometria sempre que o formato permitir.
- Converter coordenadas e unidades quando necessário.
- Triangular a geometria quando exigido pela representação interna.
- Apresentar avisos para atributos, componentes ou materiais que não possam ser preservados.
- Validar individualmente cada exportador e as capacidades da versão de Assimp integrada.
- Exibir progresso e permitir cancelamento seguro em operações longas.

Não prometer round-trip sem perda entre formatos com capacidades diferentes.

### 2.13 Salvamento, projetos e recuperação

Salvamento manual e automático configurável. Opções de autosave: desativado, 1, 3, 5, 10 ou 15 minutos. Indicador de alterações não salvas.

A gravação deve preservar o arquivo anterior até a nova gravação estar completa: temporário exclusivo no mesmo diretório, escrita integral, flush/sincronização, fechamento e rename atômico quando suportado. Falhas não podem truncar o projeto válido anterior.

Recuperação local após encerramento inesperado; oferecer ao usuário a possibilidade de recuperar a versão encontrada. O histórico completo de Undo/Redo não precisa ser persistido no arquivo principal.

### 2.14 Desempenho

Não haverá limite artificial fixo de vértices ou faces. O limite prático será determinado por memória, capacidade do dispositivo e estabilidade. O editor deverá:
- Monitorar pressão de memória e avisar antes de operações de risco.
- Atualizar buffers GPU incrementalmente.
- Evitar cópias integrais desnecessárias.
- Processar operações demoradas fora da UI e da render thread.
- Liberar recursos ao remover objetos/fechar projetos.
- Implementar Frustum Culling.
- Limitar operações concorrentes sobre a mesma malha.
- Não iniciar sistemas complexos de LOD ou streaming de geometria na 1.0.

## 3. Arquitetura técnica

### 3.1 Camadas e módulos

**Android / Java ou Kotlin**
- Interface, menus, diálogos, configurações, ciclo de vida e SAF.
- A decisão entre Java, Kotlin ou ambos permanece para a fase de implementação; JNI é obrigatório para a comunicação com o backend nativo.

**Backend C++**
- `Core`: ciclo de vida, serviços comuns e coordenação.
- `Device`: GPU, propriedades/limites Vulkan, extensões, formatos, seleção e diagnóstico do dispositivo.
- `Graphics`: Vulkan, renderização, recursos e shaders.
- `Geometry`: dados de malha, Half-Edge Mesh e operações de modelagem.
- `Scene`: Scene Graph, transformações, câmeras e luzes.
- `ImportExport`: Assimp e conversão.
- `Project`: leitura, gravação, versão e verificações de integridade X3D.
- `Editor`: seleção, ferramentas e comandos.
- `Logger`: logs centralizados.
- `Diagnostics`: diagnóstico e Crash Reports locais.

### 3.2 Vulkan e pipeline

- API mínima: Vulkan 1.1.
- Shaders-fonte: GLSL; distribuição/execução: SPIR-V (`.spv`).
- Renderização em thread dedicada.
- Render passes/etapas lógicas: Geometry, Lighting, Grid, Gizmo e Selection; a implementação pode usar subpasses ou passes separados conforme compatibilidade e custo real.
- Gerenciador centralizado de recursos Vulkan: buffers, imagens, shaders, pipelines, descriptor sets e memória.
- Geometria CPU editável separada dos buffers GPU de renderização.
- Dados: posição float32×3, normal float32×3, UV float32×2 (preparado para futuro) e índices uint32.
- Topologia editável: Half-Edge Mesh. A representação de renderização pode ser triangulada e derivada da topologia.
- Sistema destro, Y para cima, frontal em Z negativo.

O ID Buffer usa identificadores inteiros de 32 bits, preferencialmente `VK_FORMAT_R32_UINT`, após verificar suporte. Os IDs persistentes de cena continuam uint64; IDs do picking são temporários e mapeados para os IDs reais. A seleção de objetos, vértices, arestas e faces usa GPU Picking. A passagem de picking deve ser independente de MSAA quando necessário para garantir leitura determinística de IDs inteiros; readback deve evitar bloquear a render thread.

Profundidade: preferir `VK_FORMAT_D24_UNORM_S8_UINT`; usar `VK_FORMAT_D32_SFLOAT` como alternativa se suportado. MSAA 4x é preferência padrão, mas deve ser negociado com as capacidades do dispositivo. Não assumir que todo formato, amostragem ou recurso opcional está presente em toda GPU Vulkan 1.1.

### 3.3 Pipeline de renderização e efeitos

Etapas lógicas:
1. Geometry Pass.
2. Lighting Pass.
3. Grid Pass.
4. Gizmo Pass.
5. Selection Pass.

Sombras: Shadow Mapping; CSM para luz direcional, mapas convencionais para Point/Spot. SSAO, FXAA, Tone Mapping e Gamma Correction são recursos previstos. Os efeitos devem ser ativáveis individualmente e agrupados nos perfis Baixo/Médio/Alto/Personalizado. A ordem exata e a combinação entre MSAA e FXAA devem ser validadas por qualidade e desempenho durante prototipagem.

### 3.4 Threads e sincronização

- Main Thread Android: interface e eventos de entrada.
- Render Thread: comandos gráficos e renderização Vulkan.
- Thread Pool C++: importação/exportação, operações de geometria e cálculos pesados.
- JNI: comandos agrupados e transferência de resultados; evitar chamadas excessivamente frequentes.

Thread Pool ajusta o número de workers aos recursos do dispositivo; não criar uma thread nova por operação. Operações devem ser canceláveis quando seguro. Uma malha não pode ser modificada simultaneamente por tarefas incompatíveis. Resultados são publicados/sincronizados com a Render Thread por mecanismo explícito de fila/eventos.

O ciclo de vida Android deve tratar pausa, retomada, recriação da superfície e perda do dispositivo Vulkan.

### 3.5 Memória

- RAII para propriedade e liberação de recursos C++.
- Memory Pool para alocações frequentes.
- Arena Allocator para dados temporários de operações.
- GPU Allocator dedicado para memória Vulkan.

As estratégias devem ser medidas antes de serem otimizadas prematuramente. Ownership, duração e sincronização devem estar explícitos.

### 3.6 Eventos, comandos e histórico

Event System centralizado, com eventos iniciais: `ObjectSelected`, `ObjectCreated`, `ObjectDeleted`, `GeometryModified`, `TransformChanged`, `ProjectSaved` e `ProjectLoaded`. Deve ser seguro entre threads e controlar o ciclo de vida dos assinantes.

Command System representa operações como `CreateObjectCommand`, `DeleteObjectCommand`, `TransformObjectCommand`, `ExtrudeCommand`, `BevelCommand` e `MirrorCommand`. Comandos devem oferecer execução e reversão quando aplicável.

Undo/Redo usa Command History, não cópias integrais de malhas por operação. O histórico é por projeto, com limite de memória configurável e descarte de comandos antigos. Comandos podem guardar dados geométricos necessários para reversão exata; não depender apenas de inversas matemáticas.

### 3.7 Logs, erros e diagnóstico

Logger compartilhado com níveis TRACE, DEBUG, INFO, WARN, ERROR e FATAL; data/hora, módulo de origem, arquivos locais, rotação e limite de armazenamento.

Erros Vulkan:
- Warning: recurso/função indisponível, continuar com fallback.
- Error: operação falhou, manter projeto aberto quando seguro.
- Fatal: falha crítica que exige interromper o motor gráfico.

Página Diagnóstico nas configurações: GPU/fabricante/modelo, versão Vulkan, driver, dados de memória reportados, perfil gráfico, últimos avisos/erros e exportação de logs.

Crash Reports locais: motivo, módulo quando identificável, stack trace C++, dados Android, GPU/driver/Vulkan e últimas operações do Logger. Não enviar relatórios automaticamente.

## 4. Recursos fora da versão 1.0

Planejados para versões futuras, sem compromisso de data:
- Texturização e edição de UV.
- Editor de materiais PBR avançado e carregamento de texturas.
- Bevel Modifier.
- Subdivision Surface.
- Animação e rigging.
- Física/simulação.
- Sculpting.
- LOD automático e streaming avançado de geometria.
- Edição colaborativa.

A biblioteca xatlas já foi considerada para UV unwrapping em discussões anteriores, mas a geração/edição de UV e o fluxo de texturização não fazem parte do escopo funcional 1.0. A integração poderá ser retomada numa versão posterior.

## 5. Compatibilidade e requisitos de dispositivo

- Requisito gráfico mínimo: suporte a Vulkan 1.1.
- Dispositivos sem Vulkan 1.1 não são compatíveis com a viewport 3D da versão 1.0.
- Detectar GPU, driver, extensões, formatos, limites e amostras suportadas.
- Recomendar perfil gráfico inicial; permitir alteração manual.
- Testar principalmente GPUs Adreno e Mali, além de outros dispositivos Vulkan disponíveis.
- Não assumir disponibilidade de recursos opcionais só pela versão da API.
- Versão mínima do Android, ABIs, NDK, SDK e requisitos de memória ainda devem ser definidos antes da configuração final do projeto de build.

## 6. Compatibilidade e evolução do formato X3D

O repositório já contém uma especificação independente do formato binário `.bin` em [data-format.md](data-format.md), com versão 1, cabeçalho, seções META/NODE/MESH/MATL/TEXR e testes em [format-v1-validation.md](format-v1-validation.md).

**Essa especificação existente permanece a fonte de verdade para os bytes da versão 1 até que uma revisão formal seja aprovada.** Este documento mestre não altera retroativamente o schema.

Há diferenças a resolver antes de congelar o formato usado pela aplicação:
- O schema v1 atual representa referências de nós por índices de coleção, enquanto a arquitetura aprovada prevê IDs persistentes uint64.
- O schema v1 atual não descreve explicitamente componentes Camera e Light, nem a pilha de modificadores.
- A geometria persistida v1 é triangulada (posições/normais/UV0/índices); Half-Edge é a estrutura topológica editável em memória e ainda precisa de uma decisão de serialização.
- A seção MATL/TEXR já prevê dados de materiais e referências de texturas, embora texturização esteja fora do escopo funcional 1.0.
- O schema v1 não inclui submeshes nem múltiplos materiais por malha.

**Ação obrigatória na fase de formato:** elaborar uma matriz de compatibilidade entre os requisitos da cena e o schema v1; decidir se os requisitos podem ser representados por extensão compatível ou se exigem uma nova versão do formato. Não alterar a fixture canônica nem os testes v1 sem versionamento e decisão explícita. Leitores devem rejeitar versões não suportadas de forma controlada e preservar a possibilidade de migração futura.

## 7. Etapas de desenvolvimento

As etapas abaixo são sequenciais por dependência. Algumas tarefas internas podem ocorrer em paralelo após os contratos de módulo estarem estáveis. Cada etapa termina com critérios verificáveis; não avançar apenas porque o código compila.

### Etapa 0 — Preparação e decisões de build

**Entregas**
- Confirmar versão mínima do Android, SDK/NDK, ABI(s), toolchain C++ e linguagem da UI (Java, Kotlin ou combinação).
- Definir estrutura de diretórios e convenções de código.
- Configurar build Android e integração CMake/NDK.
- Definir política de dependências e versões de Assimp.
- Criar CI básico e documentação de compilação.

**Aceitação**
- Aplicativo Android vazio instala e inicia em dispositivo compatível.
- Biblioteca nativa compila para cada ABI selecionada.
- Dependências e versões estão fixadas e reproduzíveis.

### Etapa 1 — Esqueleto do aplicativo e JNI

**Entregas**
- Activity principal em landscape.
- Navegação básica e tema escuro.
- Biblioteca C++ carregada via JNI.
- Contrato inicial de comandos/erros entre UI e backend.
- Tratamento do ciclo de vida Android.

**Aceitação**
- UI não bloqueia ao chamar operações nativas de teste.
- JNI lida com erros sem crash.
- Pausa/retomada não deixa threads ou recursos órfãos.

### Etapa 2 — Core, Logger, Device e diagnóstico inicial

**Entregas**
- Módulos Core, Logger e Device.
- Coleta de propriedades Vulkan e identificação de GPU/driver.
- Tratamento de Warning/Error/Fatal.
- Logs locais com rotação.
- Primeira tela de diagnóstico.

**Aceitação**
- Dispositivo sem Vulkan 1.1 recebe mensagem clara e não tenta iniciar viewport incompatível.
- Propriedades e limitações são registradas.
- Erros simulados são classificados corretamente.

### Etapa 3 — Fundação Vulkan e viewport mínima

**Entregas**
- Instância, dispositivo lógico, filas, swapchain e sincronização.
- Render Thread dedicada.
- Gestão de superfície e recriação após ciclo de vida.
- Carregamento de SPIR-V.
- Geometry Pass mínimo e desenho de uma primitiva.
- Gerenciador de recursos Vulkan e Depth Buffer.

**Aceitação**
- Renderizar continuamente sem bloquear a UI.
- Redimensionamento/recriação da superfície funciona.
- Recursos são liberados sem vazamentos conhecidos.
- Fallbacks de formatos/amostras são testados.

### Etapa 4 — Scene Graph e transformações

**Entregas**
- Nós, IDs, componentes e hierarquia.
- Transformações locais e globais com quaternion.
- Dirty Flags.
- Câmera orbital, Perspective/Orthographic e gizmo XYZ.
- Câmeras e luzes de cena básicas.

**Aceitação**
- Hierarquias profundas e alterações de pai atualizam matrizes corretamente.
- Não há ciclos ou referências inválidas.
- Gestos de órbita, zoom e pan funcionam em tela touch.

### Etapa 5 — Geometria e topologia

**Entregas**
- Mesh CPU e buffers GPU separados.
- Half-Edge Mesh.
- Conversão de topologia para buffers triangulados.
- Normais, índices e atributos.
- Atualização incremental de buffers.

**Aceitação**
- Criar, editar e renderizar malhas de teste.
- Validar conectividade e invariantes topológicos após operações.
- Alterar parte da malha não exige reconstrução completa sem necessidade.

### Etapa 6 — Seleção, ferramentas e Command History

**Entregas**
- GPU Picking de objetos e componentes com ID Buffer inteiro.
- Seleção individual, múltipla e retangular.
- Modos Object/Vertex/Edge/Face.
- Oito ferramentas de modelagem.
- Preview temporário e confirmação/cancelamento.
- Command System e Undo/Redo por projeto.

**Aceitação**
- Seleção corresponde ao elemento sob o toque, inclusive com oclusão.
- Seleção retangular segue as regras esquerda→direita/direita→esquerda.
- Cada operação confirmada pode ser desfeita/refeita corretamente.
- Cancelar preview não altera a malha original.

### Etapa 7 — Modificadores

**Entregas**
- Modifier Stack.
- Mirror não destrutivo em X/Y/Z.
- Clipping e soldagem no plano.
- Ativar, desativar, reordenar e aplicar.

**Aceitação**
- Alterar a malha original atualiza o resultado espelhado.
- Aplicar Mirror produz geometria equivalente ao preview.
- Undo/Redo cobre edição e aplicação do modificador.

### Etapa 8 — Projetos e formato X3D

**Entregas**
- Matriz de compatibilidade de requisitos versus schema v1.
- Decisão documentada sobre manter/extender v1 ou criar nova versão.
- Leitor/gravador e migração, se aplicável.
- Validação estrutural e referências.
- Salvamento atômico e recuperação.
- Miniaturas e estrutura de diretórios do projeto.

**Aceitação**
- Fixtures e testes da versão suportada passam.
- Round-trip preserva dados dentro das regras do formato.
- Falhas de gravação preservam o arquivo anterior.
- Projeto interrompido pode ser recuperado sem sobrescrever silenciosamente o original.

### Etapa 9 — Importação e exportação

**Entregas**
- Integração Assimp.
- glTF/GLB, FBX, OBJ e DAE.
- Mapeamento de cena, geometria, materiais básicos e coordenadas.
- Progresso, cancelamento e avisos de perda.

**Aceitação**
- Corpus de arquivos de teste importa e exporta nos formatos declarados.
- Erros e atributos não suportados são informados.
- Testes de round-trip documentam perdas esperadas por formato.

### Etapa 10 — Interface completa e fluxo de trabalho

**Entregas**
- Gerenciador de projetos.
- Painéis, hierarquia, ferramentas, barra contextual e configurações.
- Unidade/precisão, Snap, eixos e preferências.
- SAF para importação/exportação.
- Autosave, recuperação e diagnósticos/Crash Reports.

**Aceitação**
- Fluxo criar → modelar → salvar → fechar → reabrir funciona.
- Preferências persistem.
- Nenhuma operação pesada bloqueia a thread principal.

### Etapa 11 — Qualidade gráfica e otimização

**Entregas**
- Solid, Wireframe, Wireframe + Solid e Flat Shading.
- Luz auxiliar de três pontos.
- Sombras, CSM, SSAO, MSAA/FXAA, Tone Mapping e Gamma Correction.
- Perfis gráficos.
- Frustum Culling e otimização de memória/transferências.

**Aceitação**
- Cada recurso opcional verifica suporte e possui fallback.
- Perfis Baixo/Médio/Alto/Personalizado são consistentes.
- Testes de desempenho e memória em GPUs diferentes documentam resultados e limitações.

### Etapa 12 — Testes integrados, estabilidade e lançamento 1.0

**Entregas**
- Testes unitários, integração, regressão e dispositivos.
- Testes prolongados de memória e renderização.
- Testes de arquivos corrompidos, interrupção e recuperação.
- Revisão de acessibilidade, tradução, licenças e créditos.
- Documentação de usuário, notas de versão e checklist de publicação.

**Aceitação**
- Nenhum bloqueador conhecido em fluxos principais.
- Sem perda de projeto nos cenários de falha cobertos.
- Compatibilidade Vulkan 1.1 verificada nos dispositivos-alvo.
- Todos os formatos anunciados passam pelo corpus de conformidade.
- Critérios de desempenho e dispositivos mínimos são publicados antes do lançamento.

## 8. Riscos e decisões que permanecem abertas

Estes itens não impedem a especificação funcional, mas devem ser resolvidos nas etapas indicadas:

1. **Build Android:** versão mínima do Android, SDK/NDK, ABIs e escolha Java/Kotlin — Etapa 0.
2. **Formato X3D:** conciliar schema v1 com IDs uint64, componentes, modificadores e topologia — Etapa 8.
3. **Vulkan:** recursos opcionais variam por GPU/driver; todos precisam de consulta e fallback — Etapas 2, 3 e 11.
4. **Half-Edge e exportação:** formatos externos podem exigir triangulação e conversão; preservar a malha editável internamente e exportar uma representação compatível.
5. **Autosave e SAF:** gravação atômica pode ter limitações diferentes em URIs SAF; definir fluxo seguro para armazenamento privado e exportação por SAF.
6. **Threading:** geometria não pode ser alterada simultaneamente por ferramentas e workers; contratos de ownership e sincronização são obrigatórios.
7. **Escopo de materiais:** somente materiais básicos na 1.0; não expor controles de texturização ainda não implementados.

## 9. Critério de conclusão da versão 1.0

A versão 1.0 estará pronta quando o usuário puder criar um projeto, importar um modelo, navegar e editar sua geometria com as ferramentas aprovadas, organizar a cena, usar Mirror, desfazer/refazer, salvar e recuperar o projeto, exportar para os formatos anunciados e trabalhar numa viewport Vulkan estável em dispositivos compatíveis.

A publicação depende de testes documentados, não apenas da conclusão das telas ou da compilação.

## 10. Documentos relacionados

- [README](../README.md)
- [Arquitetura tecnológica atual](architecture.md)
- [Formato binário X3D v1](data-format.md)
- [Testes de conformidade X3D v1](format-v1-validation.md)

---

**Nota de governança:** alterações de escopo devem atualizar este documento. Alterações no layout binário devem atualizar a especificação do formato, sua versão e as fixtures/testes correspondentes. O status de cada etapa deve ser atualizado somente após evidência de implementação e critérios de aceitação atendidos.
