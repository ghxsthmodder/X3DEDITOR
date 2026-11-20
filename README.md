# X3D Editor

**Editor de modelagem 3D para Android, com backend C++ e renderização Vulkan.**

O X3D Editor é um projeto em fase de especificação e planejamento da versão 1.0. Seu objetivo é oferecer ferramentas de modelagem poligonal, organização de cenas, importação/exportação e um formato binário nativo, priorizando desempenho, estabilidade e uso eficiente de memória em dispositivos móveis.

## Especificação principal

A especificação técnica mestre reúne o escopo funcional, arquitetura, requisitos de renderização, compatibilidade, riscos e etapas de desenvolvimento:

- **[X3D Editor 1.0 — Especificação técnica mestre e plano de desenvolvimento](docs/specification-v1.0.md)**

Esse documento é a referência de escopo da versão 1.0. Ele não significa que os recursos já estejam implementados.

## Plataforma e arquitetura

- **Plataforma:** Android exclusivo, orientação horizontal (*landscape*).
- **Interface:** Java e/ou Kotlin; a escolha final será feita na etapa de preparação do build.
- **Backend:** C++.
- **Integração:** JNI.
- **Renderização:** Vulkan 1.1.
- **Shaders:** GLSL como fonte e SPIR-V como formato compilado.
- **Importação/exportação:** Assimp.
- **Estrutura topológica editável:** Half-Edge Mesh.
- **Formato nativo:** X3D binário, extensão `.bin`.

## Escopo resumido da versão 1.0

- Gerenciador de projetos e viewport 3D.
- Modelagem poligonal: Extrude, Inset, Bevel, Loop Cut, Merge, Subdivide, Delete e Move Pivot.
- Seleção de objetos, vértices, arestas e faces.
- Transformações, Snap e restrição por eixos.
- Scene Graph com malhas, Empty, câmeras e luzes.
- Modifier Stack com Mirror.
- Undo/Redo, salvamento automático e recuperação.
- Importação/exportação glTF/GLB, FBX, OBJ e DAE.
- Diagnóstico, logs e Crash Reports locais.

Texturização, edição UV, materiais PBR avançados, animação, rigging, Bevel Modifier e Subdivision Surface ficam para versões futuras. Consulte a especificação mestre para o escopo completo e as exclusões.

## Documentação técnica

- [Especificação técnica mestre e roadmap 1.0](docs/specification-v1.0.md)
- [Arquitetura tecnológica](docs/architecture.md)
- [Formato binário interno X3D v1](docs/data-format.md)
- [Testes de conformidade do formato X3D v1](docs/format-v1-validation.md)

**Importante sobre o formato:** a especificação binária v1 existente continua sendo a fonte de verdade para o layout atual. A especificação mestre registra diferenças entre o schema atual e os requisitos funcionais aprovados; qualquer evolução deverá ser versionada e acompanhada de fixtures e testes.

## Etapas de desenvolvimento

0. Preparação e decisões de build.
1. Esqueleto do aplicativo Android e JNI.
2. Core, Logger, Device e diagnóstico inicial.
3. Fundação Vulkan e viewport mínima.
4. Scene Graph e transformações.
5. Geometria e topologia.
6. Seleção, ferramentas e Command History.
7. Modifier Stack e Mirror.
8. Projetos e formato X3D.
9. Importação e exportação.
10. Interface completa e fluxo de trabalho.
11. Qualidade gráfica e otimização.
12. Testes integrados, estabilidade e lançamento 1.0.

Os entregáveis e critérios de aceitação de cada etapa estão descritos na [especificação mestre](docs/specification-v1.0.md).

## Status

**Em planejamento — escopo funcional da versão 1.0 consolidado; implementação e validação técnica pendentes.**

O status de implementação deve ser atualizado somente quando houver código e testes que comprovem a conclusão dos critérios de aceitação.
