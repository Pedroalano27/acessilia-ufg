# Verificação do Gestor — 11/10/2026

Versão testada: `acessilia-ufg` em `296ffcd` (10/10). Ambiente: Gestor local (WSL, Node 22), usuários do seed, **sem Core rodando** — ou seja, o cenário "Core indisponível". Não testei contra um Core real nem na VM3.

Reproduzir: `C:\Users\pedro\wt\qa-web\func_1011.py` (testes na aplicação) e `C:\Users\pedro\wt\gestor-cov.sh` (cobertura).

## Resultado

| Verificação | Resultado |
|---|---|
| Suíte automática | 48 de 48 passam (eram 43) |
| Cobertura de linhas | 77% no total, mas com buracos nas partes novas (abaixo) |
| Testes na aplicação | 6 defeitos funcionais confirmados |

## Defeitos confirmados na aplicação

### 1. Audiodescrição falsa em todo material — grave
- **O que acontece:** a tela de detalhes e o HTML baixado trazem sempre o mesmo texto ("A imagem apresenta o logotipo institucional…"), com selo de "IA + Acessilia". Apareceu nos 6 materiais testados, na tela e no arquivo baixado, para admin e professor.
- **Impacto:** o aluno recebe a descrição de uma imagem que não existe no material.
- **Onde:** `src/views/materials/detail.ejs:236` e `src/routes/web/materials.routes.js:313`.
- **Correção sugerida:** só mostrar o card quando houver audiodescrição real vinda do Core; sem ela, não mostrar nada.

### 2. Envio devolve erro, mas o material é criado e fica preso — grave
- **O que acontece:** com o Core fora do ar, o envio responde 400 com um erro cru do banco (`CHECK constraint failed: current_status…`). Mesmo assim o material é criado e fica para sempre em "aguardando processamento", com o job marcado como falho.
- **Impacto:** o professor vê erro, tenta de novo e gera duplicatas; os materiais presos nunca saem da fila.
- **Onde:** o status `FALHA_PROCESSAMENTO` não está na lista aceita pelo banco (`database/migrations/001_initial_schema.sql:241`). É a mesma pendência da rodada de 29/09.
- **Correção sugerida:** migração incluindo o status; mensagem amigável ("o processamento falhou, tente novamente"); botão de reprocessar.

### 3. Envio sem arquivo é aceito
- **O que acontece:** mandar só título e disciplina cria um material cujo "arquivo" é um texto de exemplo (`PDF_SAMPLE_DATA`). Arquivo vazio (0 bytes) também é aceito.
- **Correção sugerida:** exigir arquivo e recusar arquivo vazio.

### 4. Formatos aceitos não batem com os do Core
- **O que acontece:** o Gestor aceita DOC, PPT, PPTX e TXT, que o Core recusa; e recusa WEBP e HTML (testados), que o Core aceita. TIFF e BMP também estão fora da lista do Gestor.
- **Impacto:** o professor envia um PPTX, o Gestor aceita e o Core rejeita depois.
- **Correção sugerida:** uma lista única de formatos, igual à do Core.
- **Observação:** a validação olha só a extensão do nome; um arquivo com conteúdo de outro tipo e nome `.jpg` passa.

### 5. Métricas inventadas no painel "metadados técnicos"
- **O que acontece:** todos os materiais mostram "PDDL, 1 página, 25 blocos, 2 min". São valores padrão escritos na tela.
- **Onde:** `src/views/materials/detail.ejs:303-321`; a rota lê um campo `accessible_content` que não existe no banco (`materials.routes.js:184`), então os dados reais nunca chegam.
- **Correção sugerida:** esconder o painel quando não houver metadados; ler os metadados do arquivo processado.

### 6. Data de publicação inválida devolve erro técnico
- **O que acontece:** uma data mal formada responde "Invalid time value".
- **Correção sugerida:** validar a data e responder em português.

## Achados por leitura de código (não exercitados)

- **Progresso ao vivo desligado:** `updateJobProgress` existe e a tela escuta o evento, mas nada chama a função.
- **Modo simulado pode ligar em produção:** o cliente do Core vira simulado se qualquer argumento do processo contiver "test"; e todo job com id iniciado em `ext_acessilia_` é dado como concluído com 98% de confiança sem consultar o Core (`src/integrations/bot-acess/client.js`).
- **Conteúdo adaptado fixo:** a "síntese do texto" na tela são três frases genéricas iguais para todo material.

## Os testes automáticos são bons?

Passam todos, mas não protegem as partes novas:

| Arquivo | Linhas cobertas | O que fica de fora |
|---|---|---|
| `integrations/bot-acess/client.js` | 44% | Toda a conversa real com o Core |
| `routes/web/materials.routes.js` | 56% | A tela de detalhes inteira e o caminho de falha do envio |
| `routes/web/monitoring.routes.js` | 20% | Quase tudo |
| `routes/web/users.routes.js` | 39% | Criar e editar usuário |
| `routes/web/academic.routes.js` | 43% | Cadastros acadêmicos |
| `routes/api/notifications.api.js` | 36% | O canal de progresso ao vivo |

- **Teste que não testa o que diz:** "Deve enviar documento com multipart document_file e source=gestor" cai no atalho de simulação e só confere que voltou "QUEUED". O formulário enviado ao Core nunca é montado no teste.
- **Nenhum teste cobre Core indisponível**, que é onde está o defeito 2.
- **Nenhum teste abre a tela de detalhes**, onde estão os defeitos 1 e 5.
- **Pouca granularidade:** os 10 arquivos de integração têm um teste grande cada; quando um falha, não dá para saber qual parte quebrou.

Testes sugeridos: envio com Core fora do ar (espera status de falha e mensagem amigável); envio sem arquivo; tela de detalhes sem audiodescrição (o card não deve aparecer); cliente do Core contra um servidor falso, conferindo os campos `document_file`, `source` e `mode`.

## Relação com o bug do mapa (Jonathan)

A imagem solta que passa só pelo Docling é defeito do Core/Toolbox. No Gestor o efeito seria um material "processado" sem audiodescrição, hoje encoberto pelo texto fixo do defeito 1. Vale retestar esse caso no Gestor quando houver um Core disponível.

---

# Rodada 2 — testes criteriosos com um Core simulado (11/10, tarde)

Subi o Gestor local junto com um "Core falso" na porta 8000, que responde como o Core e registra tudo o que recebe. Foram 95 verificações em sequência. Roteiros: `C:\Users\pedro\wt\qa-web\round2_1011.py` e `round2b_1011.py`; saídas em `round2_1011.log` e `round2b_1011.log`; Core falso em `fake_core.py`.

Limite desta rodada: o Core é simulado. O comportamento do Core real (o que ele devolve, se chama o aviso de conclusão) não foi testado.

## Defeitos graves

### R2-1. O arquivo do professor não chega ao Core
- **O que acontece:** enviei um JPG de 93.925 bytes. O Core recebeu 10 bytes com o texto `PDF_SAMPLE`, nos campos `document_file` e `file`.
- **Causa:** `dispatchJob` não passa o conteúdo do arquivo ao cliente (`src/integrations/bot-acess/service.js:14`), e o cliente usa um conteúdo de exemplo quando não recebe nada (`client.js`, `fileBuffer || Buffer.from('PDF_SAMPLE')`).
- **Impacto:** nenhum material é de fato processado; o Core trabalha sobre um arquivo de exemplo.
- **Correção sugerida:** ler o arquivo salvo e enviá-lo; falhar se não houver arquivo, em vez de mandar exemplo.

### R2-2. O material nunca sai de "processando" sozinho
- **O que acontece:** depois do envio, o Gestor não consulta o Core nenhuma vez (0 consultas em 6 s, com o Core respondendo "concluído"). O material só avança se alguém chamar o aviso de conclusão do Gestor.
- **Correção sugerida:** consultar o status periodicamente ou confirmar que o Core real chama o aviso de conclusão no formato que o Gestor espera.

### R2-3. O que o aluno baixa não vem do Core
- **O que acontece:** nenhum download foi pedido ao Core. O HTML entregue é montado pelo Gestor com o título, a descrição e a audiodescrição fixa. "PDF", "DOCX" e qualquer formato inventado devolvem um texto simples com o selo. "MP3" redireciona para um áudio de amostra (`/public/audio/sample_acessivel.mp3`).
- **Impacto:** o material "acessibilizado" não contém o conteúdo do documento.
- **Correção sugerida:** buscar o arquivo processado no Core e entregá-lo; recusar formato desconhecido.

### R2-4. Selo de conformidade emitido sem processamento
- **O que acontece:** o material acima, que o Core nunca recebeu, saiu com "DOCUMENTO REVISADO E APROVADO… WCAG 2.2 AAA", certificado público com QR Code, e o comparador de versões listando "contraste validado ≥ 7:1", "síntese de fala gerada em MP3", "tabelas com cabeçalhos". Essa lista é texto fixo.
- **Correção sugerida:** só afirmar o que foi verificado; tirar a lista fixa do comparador.

### R2-5. Página de monitoramento quebrada
- **O que acontece:** `/monitoramento` responde erro 500 sempre: `no such table: conversion_jobs`.
- **Correção sugerida:** usar a tabela `processing_jobs` ou criar a migração que falta.

## Fluxo de revisão

| Verificação | Resultado |
|---|---|
| Iniciar a mesma revisão duas vezes | Cria 2 revisões abertas |
| Pedir correção sem justificativa | Aceito |
| Concluir de novo uma revisão já concluída | Aceito: de "correção solicitada" foi para "aprovado" e depois para "reprovado" |
| Concluir revisão inexistente | Erro técnico: `Cannot read properties of undefined` |
| Status de revisão inválido | Recusado corretamente |
| Aluno baixar antes de aprovar | Bloqueado corretamente |
| Professor aprovar o próprio material | Bloqueado corretamente |

Correção sugerida: revisão concluída não pode ser concluída de novo; exigir justificativa ao pedir correção ou reprovar; impedir segunda revisão aberta da mesma versão.

## Falhas do Core

| Cenário | Resultado |
|---|---|
| Core recusa o formato (PPTX) | Professor vê erro cru do banco; material criado e preso em "aguardando processamento" |
| Core avisa que falhou | Material vira "correção solicitada", como se o professor tivesse errado |
| Aviso com status desconhecido | Tratado como falha, sem recusar |
| Aviso de conclusão repetido | Tratado corretamente (não duplica versão) |
| Reprocessar em lote | "Ação em lote não suportada" — não há como reprocessar |

## Validação de entradas

| Entrada | Resultado |
|---|---|
| Título só com espaços | Aceito |
| Título de 5.000 caracteres | Aceito |
| Disciplina inexistente | Erro cru: `FOREIGN KEY constraint failed` |
| Categoria inválida | Erro cru: `CHECK constraint failed: category…` |
| Solicitação para material inexistente | Erro 500 |
| Solicitação com prioridade inválida | Erro 500 |
| Mudar solicitação para status inválido ou vazio | Erro 500 |
| Mudar status de solicitação inexistente | Responde como se tivesse funcionado |
| Mesma solicitação enviada 3 vezes | 3 solicitações abertas |
| Usuário com e-mail repetido | Erro cru: `UNIQUE constraint failed: users.email` |
| Usuário com e-mail inválido ("isto-nao-e-email") | Aceito |
| Usuário com nome só de espaços | Aceito |
| Usuário com função inexistente | Erro cru, e o usuário fica gravado sem função |
| Feedback com tipo inválido | Erro cru do banco |
| Editar texto alternativo inexistente | Responde como se tivesse salvo |
| Arquivo de 60 MB | Recusado, mas com mensagem técnica em inglês |

Correção sugerida comum: validar antes de gravar e responder em português; criar usuário e função numa transação só.

## O que funcionou bem

- Login dos quatro perfis; aluno não consegue enviar material.
- 14 endereços com identificador inválido ou inexistente respondem 404 sem erro técnico.
- Turma de outro professor é recusada com mensagem clara.
- Arquivo de 12 MB e nome com acento e espaço são aceitos.
- 12 envios simultâneos: todos criados, servidor continuou de pé.
- Relatório CSV de materiais bate com o banco (40 linhas, 40 materiais).
- Preferência de acessibilidade inválida é ignorada.
- Limite de 100 requisições por minuto está ativo.

## Observações

- O limite de requisições também vale para `/healthz`; um monitor externo pode receber 429 num pico.
- O relatório de conformidade LBI mostra "100%"; não conferi de onde vem o número.
- O aluno de teste não baixou o material aprovado porque o material não tinha turma; não é defeito.
- Pontos de segurança observados ficaram fora deste relatório, como combinado; estão nas saídas locais dos roteiros.

---

# Rodada 3 — interface no navegador (11/10, fim da tarde)

Gestor local aberto no navegador, perfil administrador. Percorri 13 telas em três larguras (1366, 768 e 375 px) e nos modos de exibição (claro, escuro, alto contraste, Slate, fonte grande, dislexia), medindo contraste, transbordo e estrutura, e conferi visualmente as telas de usuários, materiais e detalhes.

Limites: só o perfil administrador; cliques reais falharam parte do tempo (a janela do navegador estava em segundo plano), então parte da interação foi acionada por script; não testei com leitor de tela nem enviei arquivo pela tela.

## Defeitos visuais

### V1. Tema escuro automático deixa botões invisíveis — grave
- **O que acontece:** quando o sistema operacional está em modo escuro, o Gestor escurece sozinho e vários textos ficam da cor do fundo: o nome "Acessilia Gestor" no menu (contraste 1,1), o botão "Sair" (1,2), o sino de notificações (1,2), "Desativar" (1,7), "Reativar" (1,2), "Visualizar" (1,8), "Exportar CSV" (2,1), "Selo Acessilia" (1,7). O mínimo recomendado é 4,5.
- **Impacto:** quem usa o computador em modo escuro não vê como sair nem os botões de ação das tabelas.
- **Observação:** no tema claro esses mesmos itens estão corretos.

### V2. Modo "alto contraste" piora o contraste dos botões — grave
- **O que acontece:** com "Contraste" ligado, os mesmos itens continuam ilegíveis ("Sair" 1,1; nome do sistema 1,0; "Selo Acessilia" 1,1; "Visualizar" 2,1; "Exportar CSV" azul sobre preto 2,2).
- **Impacto:** é o modo feito para baixa visão, e é onde os botões somem.
- **O modo Slate** tem o mesmo problema ("Visualizar" 1,6; "Sair" 1,4).

### V3. Tela de detalhes transborda no celular
- **O que acontece:** em 375 px a página de detalhes do material ganha rolagem horizontal de cerca de 106 px (255 px com fonte grande). Quem causa é a barra de controles do "Leitor Acessível" (botões de tema e "Reset").
- As outras 7 telas testadas não transbordam.

### V4. Cabeçalho ocupa um terço da tela no celular
- **O que acontece:** a barra de acessibilidade (8 botões), o sino, o perfil e "Sair" somam 293 px de altura em um aparelho de 812 px, em todas as telas.
- **Correção sugerida:** recolher os controles num menu.

### V5. Textos quebrados no meio da palavra
- "Desativar" aparece como "Desativ / ar" na tabela de usuários.
- O selo aparece como "SELO / ACESSILI / A" no celular.
- As etiquetas de função ("ADMINISTRADOR DO SISTEMA", "REVISOR DE ACESSIBILIDADE") quebram em duas linhas com o fundo recortado.
- Com fonte grande e alto contraste, os títulos das colunas da tabela de materiais aparecem uma letra por linha (visto em tela estreita).

## Defeitos de funcionamento na interface

- **Busca sem resultado não avisa:** pesquisar algo inexistente deixa a tabela vazia, sem mensagem.
- **Filtro de status incompleto:** a tabela tem materiais em "CORRECAO_SOLICITADA", mas o filtro não oferece essa opção, nem "reprovado".
- **Status em código:** aparecem como `AGUARDANDO_PROCESSAMENTO`, `CORRECAO_SOLICITADA`, em vez de texto legível.
- **Sem paginação:** os 52 materiais vêm todos numa página só.
- **Formulário de envio:** o campo de arquivo não é obrigatório (é a origem do "envio sem arquivo"); não informa tamanho máximo nem formatos; com o diálogo aberto, os 9 links do menu atrás continuam alcançáveis pelo teclado.
- **Fechar com Esc:** o diálogo não fechou com Esc no meu teste por script; falta confirmar com teclado real.
- **Seletores de status em Solicitações** não têm rótulo para leitor de tela (4 ocorrências).
- **Usuário sem nome** aparece como linha em branco na tabela (é o usuário de "nome só com espaços" da rodada 2).

## Os testes do projeto sujam o banco de desenvolvimento

- **O que acontece:** a suíte automática grava no mesmo banco usado pela aplicação. Depois de rodá-la, o banco local ficou com disciplinas repetidas ("Sistemas Embarcados" três vezes no seletor), dezenas de materiais de teste e as preferências do administrador trocadas para alto contraste, fonte muito grande e dislexia.
- **Correção sugerida:** os testes devem usar um banco temporário próprio.

## O que está bom

- No tema claro, o contraste dos itens medidos está correto.
- Todas as telas têm um único título principal, idioma declarado, imagens com texto alternativo e nenhum botão sem nome.
- Nenhuma tela mostrou "undefined", "NaN" ou data inválida.
- O diálogo de envio declara papel e título para leitor de tela e leva o foco para o primeiro campo.
- No celular o menu lateral recolhe e 7 das 8 telas não transbordam.

---

# Rodada 4 — interface por perfil: professor, revisor e aluno (11/10, noite)

Entrei com cada usuário de teste no navegador e passei o mesmo roteiro em todos: menu, painel, lista de materiais, três telas de detalhes (computador e celular), todas as rotas, telas próprias do perfil, teclado e preferências. Roteiro: `C:\Users\pedro\wt\qa-web\role_audit.js`.

Limites: interação acionada por script dentro da página (cliques reais continuaram instáveis com a janela em segundo plano); sem leitor de tela; sem envio de arquivo pela tela.

## Defeitos que valem para todos os perfis

### P1. Não existe botão de download — grave
- **O que acontece:** nenhuma tela tem link ou botão para baixar o material. A rota `/materiais/:id/download` existe, mas nada na interface aponta para ela (conferido também no código das telas).
- **Impacto:** o aluno não tem como baixar o material acessível pela interface.

### P2. Tela de detalhes mostra "documento acessibilizado" em qualquer estado — grave
- **O que acontece:** materiais em "processando", "enviado" e "correção solicitada" exibem a seção "Documento Acessibilizado pelo Acessilia", o botão "Ver Certificado", o audiolivro e a audiodescrição, iguais aos de um material aprovado.
- **Correção sugerida:** mostrar essas seções só depois do processamento e da aprovação.

### P3. Conteúdo de demonstração fixo em todo material
- A mesma equação (`∫ sin(x)/x dx = π/2`) com "Ouvir Equação Falada" e "Copiar LaTeX".
- O mesmo audiolivro de amostra (`/public/audio/sample_acessivel.mp3`).
- Uma imagem quebrada: a figura "Logotipo institucional processado pelo Acessilia Core" não tem endereço (`src=""`, `detail.ejs:247`).
- Somam-se à audiodescrição fixa já relatada.

### P4. Tela de detalhes transborda no celular para todos os perfis
- Rolagem lateral de 121 px nos 9 materiais conferidos.

### P5. Botões de acessibilidade não informam se estão ligados
- Os 8 botões da barra (contraste, dislexia, fonte etc.) não têm `aria-pressed`; o leitor de tela não diz se o modo está ativo.

## Professor

- A fila de revisões aparece para o professor, mas sem nenhuma ação; e o professor vê os seletores para mudar o status das solicitações dos alunos. Vale confirmar se é o desejado.
- Título de 5.000 caracteres aparece inteiro na tabela de materiais e no seletor de solicitações (é o material de teste da rodada 2).
- Caixas de seleção em lote: 52, todas com rótulo. Correto.

## Revisor

- **Contagens que não batem:** o painel diz "Revisões pendentes 3" e "Em análise 1"; a tela de revisões diz "Fila de Materiais para Análise (5)".
- **"Histórico de Análises Concluídas" lista revisão não concluída** (com decisão "EM_ANDAMENTO").
- **Material em "correção solicitada" fica na fila do revisor** sem ação possível, só "Ver Arquivo".
- A tela de materiais do revisor carrega o formulário de envio escondido, embora ele não possa enviar.

## Aluno

- **Sem acesso a "Solicitações" pelo menu.** O painel mostra "Minhas Solicitações 4" e a página existe, mas o menu do aluno só tem Dashboard, Cursos, Disciplinas e Materiais.
- **Ferramenta de revisor aparece para o aluno.** No material 25 o aluno vê "Validar e Aprovar Alt Text" e "Salvar Ajuste".
- **Lista de materiais do formulário de solicitação é maior que a do aluno.** Ele vê 8 materiais na lista, mas o seletor de "Nova Solicitação" traz mais de 30 títulos, incluindo materiais não aprovados e uma opção em branco.
- **Formulário "Salvar Curso" carregado escondido** na tela de cursos do aluno.
- **Página de acesso negado mostra código interno:** "não possui a permissão necessária (usuarios.visualizar)".

## O que funcionou bem nos três perfis

- Menus diferentes por perfil e página de acesso negado amigável nas 5 rotas restritas.
- Lista e CSV do aluno trazem só os 8 materiais dele.
- Material com publicação agendada: o que o aluno vê já tinha a data vencida; correto.
- Link "Pular para o conteúdo principal" existe e funciona; nenhuma ordem de tabulação forçada; regra de foco visível presente no CSS.
- Atalhos: `?` abre a ajuda com o foco dentro, Esc fecha, Alt+C alterna o contraste.
- Preferência de dislexia é salva e volta depois de recarregar.
- Nenhuma tela mostrou "undefined", "NaN" ou data inválida.
