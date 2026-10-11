# RELATÓRIO DE QA — Acessilia Gestor
## A11yDevs / `acessilia-ufg` · branch `main` · commit `296ffcd` (10/10/2026)

**Homologador:** Pedro (tester/QA) — execução com apoio de agente (Claude Code); achados reproduzidos por roteiro e conferidos no código
**Data:** 11/10/2026
**Escopo:** as três entregas desde 26/09 (card de audiodescrição, integração com o Core, documentação) e o fluxo completo de material
**Rodada anterior:** 29/09/2026 (43/43 testes; pendência do status de falha)

---

## 1. Objetivo

Verificar se o Gestor, na versão de 10/10, cumpre o ciclo do material acessível — envio, processamento no Core, revisão, aprovação e entrega ao aluno — e se a interface está utilizável nos quatro perfis.

---

## 2. Veredito

**Não aprovado para homologação.** A suíte automática passa (48 de 48), mas o ciclo principal não funciona de ponta a ponta:

- o arquivo do professor não chega ao Core;
- o material não avança sozinho depois de processado;
- o que seria entregue ao aluno é montado pelo próprio Gestor, com conteúdo de demonstração;
- não há botão de download em nenhuma tela.

Parte disso pode ser funcionalidade ainda não implementada, e não defeito. Mesmo assim, hoje a tela afirma coisas que não aconteceram (audiodescrição "por IA", selo WCAG 2.2 AAA, certificado).

| Gravidade | Quantidade |
|---|---|
| Grave | 10 |
| Média | 14 |
| Baixa | 13 |

---

## 3. Ambiente

| Item | Detalhe |
|------|---------|
| Onde rodou | Máquina local, Windows 11 com WSL (Ubuntu) |
| Node | 22.23.3 |
| Gestor | `acessilia-ufg` em `296ffcd`, banco SQLite local com dados do seed |
| Usuários | os quatro do seed (administrador, professor, revisor, aluno) |
| Core | **simulado** (`fake_core.py`, porta 8000): responde como o Core e registra o que recebe |
| Navegador | navegador embutido, 1366, 768 e 375 px; temas claro, escuro, alto contraste, Slate, fonte grande, dislexia |
| Roteiros | `C:\Users\pedro\wt\qa-web\` (`func_1011.py`, `round2_1011.py`, `round2b_1011.py`, `role_audit.js`) |

> [!NOTE]
> **Core simulado.** Nenhum teste foi feito contra o Core real nem na VM3. O que o Core real devolve, e se ele avisa o Gestor ao terminar, continua sem verificação.

> [!NOTE]
> **Interface.** Os cliques reais ficaram instáveis (janela em segundo plano); parte da interação foi acionada por script dentro da página. Não houve teste com leitor de tela nem envio de arquivo pela tela.

---

## 4. O que foi executado

| Rodada | O que | Volume |
|---|---|---|
| 1 | Suíte automática do projeto, com medição de cobertura | 48 testes |
| 2 | Aplicação rodando, sem Core e depois com Core simulado: ciclo completo, entradas inválidas, identificadores errados, simultaneidade | 95 verificações |
| 3 | Interface como administrador: 13 telas, 3 larguras, 6 modos de exibição | contraste, transbordo, estrutura |
| 4 | Interface como professor, revisor e aluno | menu, painel, lista, detalhes, rotas, teclado, preferências |

---

## 5. Achados graves

| ID | Achado | Evidência | Correção sugerida |
|---|---|---|---|
| G1 | **O arquivo não chega ao Core.** | JPG de 93.925 bytes enviado; o Core recebeu 10 bytes com o texto `PDF_SAMPLE`. `service.js:14` não passa o conteúdo; `client.js` usa um exemplo quando não recebe nada. | Ler o arquivo salvo e enviá-lo; falhar se não houver arquivo. |
| G2 | **O material nunca sai de "processando" sozinho.** | 0 consultas de status ao Core, mesmo com ele respondendo "concluído". | Consultar o status periodicamente, ou confirmar que o Core real chama o aviso de conclusão no formato esperado. |
| G3 | **O material entregue não vem do Core.** | 0 pedidos de download ao Core. O HTML é montado pelo Gestor; "PDF" e "DOCX" devolvem um texto simples; "MP3" redireciona para `/public/audio/sample_acessivel.mp3`. | Buscar o arquivo processado no Core; recusar formato desconhecido. |
| G4 | **Não existe botão de download.** | Nenhuma tela aponta para `/materiais/:id/download` (conferido nas telas e no código das views). | Incluir os botões de download na tela de detalhes. |
| G5 | **Audiodescrição falsa em todo material.** | O mesmo texto sobre um "logotipo institucional", com selo "IA + Acessilia", na tela e no HTML baixado (`detail.ejs:236`, `materials.routes.js:313`). | Mostrar o card só quando houver descrição real do Core. |
| G6 | **Selo e certificado emitidos sem processamento.** | Material que o Core nunca recebeu saiu "aprovado, WCAG 2.2 AAA", com certificado e QR Code; o comparador de versões lista conferências fixas ("contraste validado ≥ 7:1", "MP3 gerado"). | Afirmar só o que foi verificado; remover a lista fixa. |
| G7 | **"Documento acessibilizado" em qualquer estado.** | Materiais em "processando", "enviado" e "correção solicitada" mostram a seção de documento acessibilizado, "Ver Certificado" e audiolivro. | Exibir essas seções só após processamento e aprovação. |
| G8 | **Envio devolve erro, mas cria o material.** | Com o Core fora do ar ou recusando o formato: resposta 400 com `CHECK constraint failed: current_status…`; material criado e preso em "aguardando processamento". O status `FALHA_PROCESSAMENTO` não existe no banco (`001_initial_schema.sql:241`). **Pendência desde 29/09.** | Migração com o status; mensagem em português; ação de reprocessar. |
| G9 | **Botões invisíveis no tema escuro e no alto contraste.** | Contraste entre 1,0 e 2,2 (mínimo 4,5) em "Sair", nome do sistema, sino, "Desativar", "Reativar", "Visualizar", "Exportar CSV", "Selo Acessilia". O tema escuro liga sozinho quando o sistema operacional está em modo escuro. No tema claro está correto. | Revisar as cores desses componentes nos três temas escuros. |
| G10 | **Página de monitoramento quebrada.** | `/monitoramento` responde 500: `no such table: conversion_jobs`. | Usar `processing_jobs` ou criar a migração. |

---

## 6. Achados médios

| ID | Achado | Evidência |
|---|---|---|
| M1 | Revisão já concluída pode ser concluída de novo | O mesmo material foi de "correção solicitada" a "aprovado" e a "reprovado" |
| M2 | A mesma revisão pode ser iniciada duas vezes | 2 revisões abertas para a mesma versão |
| M3 | Pedir correção sem justificativa é aceito | Resposta 302, status alterado |
| M4 | Falha do Core vira "correção solicitada" | O professor recebe o material como se o erro fosse dele; não há reprocessar |
| M5 | Envio sem arquivo é aceito | Cria material com `PDF_SAMPLE_DATA`; arquivo de 0 bytes também passa; o campo não é obrigatório na tela |
| M6 | Formatos não batem com os do Core | Aceita DOC, PPT, PPTX, TXT (o Core recusa); recusa WEBP e HTML (o Core aceita) |
| M7 | Métricas inventadas | Todo material mostra "PDDL, 1 página, 25 blocos, 2 min"; a rota lê um campo que não existe (`accessible_content`) |
| M8 | Conteúdo de demonstração fixo | Mesma equação, mesmo audiolivro de amostra, três frases genéricas de "síntese" e uma imagem sem endereço (`detail.ejs:247`) |
| M9 | Erros 500 em Solicitações | Material inexistente, prioridade inválida, status inválido ou vazio |
| M10 | Usuário gravado pela metade | Função inexistente: erro, mas o usuário fica no banco sem função |
| M11 | Aluno sem "Solicitações" no menu | O painel mostra "Minhas Solicitações 4" e a página existe |
| M12 | Ferramenta de revisor aparece para o aluno | "Validar e Aprovar Alt Text" e "Salvar Ajuste" no material 25 |
| M13 | Detalhes transborda no celular | 121 px de rolagem lateral em todos os materiais conferidos; causa: controles do "Leitor Acessível" |
| M14 | A suíte automática suja o banco da aplicação | Disciplinas repetidas, dezenas de materiais de teste, preferências do administrador trocadas |

---

## 7. Achados baixos

| ID | Achado |
|---|---|
| B1 | Erros crus do banco na tela: disciplina inexistente, categoria inválida, e-mail repetido, tipo de feedback inválido |
| B2 | Aceita título só com espaços, título de 5.000 caracteres, e-mail sem formato de e-mail, nome só com espaços |
| B3 | Mesma solicitação enviada três vezes gera três registros |
| B4 | Mudar status de solicitação inexistente e editar texto alternativo inexistente respondem como se tivessem funcionado |
| B5 | Data de publicação inválida devolve "Invalid time value"; arquivo de 60 MB devolve mensagem técnica em inglês |
| B6 | Concluir revisão inexistente devolve `Cannot read properties of undefined` |
| B7 | Busca sem resultado deixa a tabela vazia, sem mensagem; não há paginação (52 materiais numa página) |
| B8 | Filtro de status sem "correção solicitada" e "reprovado"; status exibidos em código (`AGUARDANDO_PROCESSAMENTO`) |
| B9 | Contagens do revisor não batem (painel: 3 pendentes + 1 em análise; tela: 5 na fila); "Histórico de Concluídas" lista revisão em andamento |
| B10 | Seletor de "Nova Solicitação" do aluno lista mais de 30 materiais (a lista dele tem 8) e uma opção em branco |
| B11 | Palavras quebradas no meio ("Desativ/ar", "SELO ACESSILI/A", etiquetas de função); cabeçalho ocupa 293 de 812 px no celular |
| B12 | Botões da barra de acessibilidade sem `aria-pressed`; seletores de status em Solicitações sem rótulo |
| B13 | Página de acesso negado mostra o código interno da permissão; progresso ao vivo existe no código mas nada o aciona |

---

## 8. Cobertura da suíte automática

48 de 48 passam; 77% das linhas no total. Os buracos estão nas partes novas:

| Arquivo | Linhas cobertas | O que fica de fora |
|---|---|---|
| `integrations/bot-acess/client.js` | 44% | Toda a conversa real com o Core |
| `routes/web/materials.routes.js` | 56% | A tela de detalhes e a falha de envio |
| `routes/web/monitoring.routes.js` | 20% | Quase tudo (por isso G10 passou) |
| `routes/web/users.routes.js` | 39% | Criar e editar usuário |
| `routes/api/notifications.api.js` | 36% | Progresso ao vivo |

- O teste "deve enviar documento com `document_file` e `source=gestor`" cai no atalho de simulação e só confere que voltou "na fila"; o formulário nunca é montado. Por isso G1 não foi pego.
- Nenhum teste cobre Core fora do ar (G8) nem abre a tela de detalhes (G5, G7).
- O modo simulado liga se qualquer argumento do processo contiver "test"; pode ligar em produção por acidente.

---

## 9. Pontos positivos

- Login dos quatro perfis, menus diferentes por perfil e página amigável de acesso negado.
- Aluno não envia material, não baixa antes da aprovação e só vê os materiais dele (lista e CSV).
- Professor não aprova o próprio material; turma de outro professor é recusada com mensagem clara.
- 14 endereços com identificador inválido respondem 404 sem erro técnico.
- 12 envios simultâneos criados; servidor continuou de pé; limite de 100 requisições por minuto ativo.
- Aviso de conclusão repetido não duplica versão.
- Tema claro com contraste correto; um título principal por tela; idioma declarado; imagens com texto alternativo.
- "Pular para o conteúdo", foco visível, atalhos (`?`, Esc, Alt+C) e preferência de dislexia persistente funcionam.

---

## 10. Fora de escopo e pendências

- Integração com o Core real e homologação na VM3.
- Leitor de tela, navegação real por teclado, envio de arquivo pela tela.
- Cadastro de cursos e disciplinas, auditoria, backup e a origem do "100%" no relatório de conformidade.
- Pontos de segurança observados: tratados em privado, fora deste relatório.

---

## 11. Como reproduzir

1. Subir o Gestor e o Core simulado: `wsl -d Ubuntu -- bash /mnt/c/Users/pedro/wt/gestor-run2.sh`
2. Ciclo e entradas: `python C:\Users\pedro\wt\qa-web\round2_1011.py` e depois `round2b_1011.py` (este respeita o limite de requisições)
3. Cobertura: `wsl -d Ubuntu -- bash /mnt/c/Users/pedro/wt/gestor-cov.sh`
4. Interface: carregar `role_audit.js` na página e chamar `run(email, senha)` com um usuário do seed

Saídas completas: `round2_1011.log`, `round2b_1011.log` e o relatório por rodada em `RELATORIO-QA-GESTOR-POR-RODADA-2026-10-11.md`.

---

## 12. Próximos passos sugeridos

1. Corrigir G1 a G4 e G8, que impedem o ciclo básico.
2. Retirar o conteúdo de demonstração das telas de material (G5 a G7, M7, M8) ou marcá-lo claramente como exemplo.
3. Reteste com os mesmos roteiros e, em seguida, com um Core real.
4. Converter um recorte dos roteiros em testes da suíte do projeto (proposta de PR à parte).
