# Auditoria completa — Luah Produção Financeiro

Data: 28/09/2026

## Resumo executivo

A aplicação já possui uma boa base funcional: cadastro de shows, séries, equipe, pagamentos, finanças, notas fiscais, relatórios, calendário, backups locais, importação/exportação e sincronização com JSONBin.

Entretanto, a auditoria encontrou problemas **críticos de segurança, sincronização e consistência financeira** que devem ser resolvidos antes de uma grande reformulação visual.

O erro **“erro ao salvar — a reenviar”** é confirmado pelo código como uma falha no envio da fila para o JSONBin. A mensagem atual não informa se foi rede, chave inválida, Bin inexistente, limite de requisições ou erro do servidor.

Também há uma fragilidade importante: a aplicação pode tentar puxar dados da nuvem depois de uma falha de envio. Isso pode misturar dados antigos, ressuscitar exclusões e limpar ou reprocessar a fila antes de a alteração local ser confirmada.

---

## Prioridade P0 — resolver antes do redesign

| Área | Problema | Impacto |
|---|---|---|
| Segurança | A API Key master do JSONBin fica no JavaScript e no `localStorage`; os Bins são criados como públicos. | Qualquer pessoa com acesso ao código/ambiente pode potencialmente ler ou alterar dados financeiros. |
| Fila | Qualquer exceção, 401, 403, 404, 429, 5xx, CORS ou timeout vira a mesma mensagem de erro. | Impossível saber o que corrigir. |
| Offline | Mesmo com `navigator.onLine=false`, o app tenta três requisições antes de deixar a fila pendente. | O usuário vê erro quando deveria ver apenas “pendente offline”. |
| Push/pull | Depois de falhar ao enviar, o app ainda pode fazer pull e merge com a nuvem. | Uma exclusão pode voltar e uma alteração local pode ser sobrescrita. |
| Polling | Existem dois blocos de polling com comportamentos diferentes. | Risco de chamadas concorrentes, estados divergentes e reenvios irregulares. |
| Backup | Falhas do `localStorage`, quota cheia ou armazenamento bloqueado são silenciosas. | A fila pode desaparecer ao recarregar sem aviso. |
| Restauração | Restaurar backup não cria uma operação explícita de publicação na nuvem. | O próximo pull pode sobrescrever a restauração. |
| NFs/caixa | NF mensal e NF individual podem coexistir para os mesmos shows; status de NF e recebimento são inconsistentes. | Risco de dupla cobrança e caixa incorreto. |

### Correção imediata recomendada para o erro de reenvio

1. Se estiver offline, **não executar `fetch`**: manter a fila e mostrar “Offline — 1 lançamento pendente”.
2. Registrar status HTTP e corpo limitado da resposta, sem registrar a API Key.
3. Diferenciar:
   - `401/403`: API Key inválida, expirada ou sem permissão;
   - `404`: Bin inexistente ou ID incorreto;
   - `429`: limite de requisições;
   - `5xx`: instabilidade do JSONBin;
   - `TypeError/CORS/DNS`: falha de rede ou bloqueio do navegador.
4. Usar timeout com `AbortController` e backoff progressivo.
5. Enquanto houver item na fila, não executar pull/merge destrutivo.
6. Remover o item da fila somente depois de confirmação bem-sucedida.
7. Unificar os dois pollings em um único `tickSync()` com mutex.
8. Criar **Desconectar/limpar conexão**, preservando os backups, para permitir rotação de credencial.

> A causa exata da ocorrência no celular ainda exige o status HTTP e o erro de rede do aparelho. O código confirma a falha genérica, mas não permite afirmar se a API Key foi rejeitada sem esse diagnóstico.

---

## Auditoria por área

### 1. Sincronização e conexão

**Problemas confirmados:**

- A fila salva o estado inteiro do banco, não uma operação pequena por registro.
- Exclusões não possuem tombstone; uma exclusão local pode voltar no merge.
- `localStorage` é usado para fila e dados sem tratamento visível de quota.
- Há dois mecanismos de polling.
- Falhas do polling são parcialmente silenciosas.
- Credenciais antigas permanecem armazenadas mesmo depois de sair.
- Não há idempotência robusta para um PUT aceito cujo retorno se perde.

**Melhorias:**

- Migrar fila para IndexedDB ou criar uma camada de armazenamento com validação de quota.
- Usar `updatedAt`, versão por registro e tombstones para exclusões.
- Criar log de sincronização com horário, status e quantidade de itens pendentes.
- Mostrar “salvo neste aparelho”, “pendente de sincronização” e “sincronizado” como estados distintos.
- Criar reconciliação de conflitos entre aparelhos.
- Considerar backend/proxy seguro, em vez de expor a API Key master no frontend.

### 2. Backups e recuperação

**Problemas confirmados:**

- A interface promete 365 dias, mas o código limita a 180 snapshots.
- O backup é feito a cada alteração e a cada 5 minutos, não uma cópia diária simples.
- O aviso de backup diário consulta uma chave diferente daquela usada pelo backup real.
- Restaurar não remove necessariamente fila/metadados antigos.
- Importação não tem versão, schema ou validação profunda.
- JSON corrompido pode resultar em banco vazio em memória.

**Melhorias:**

- Definir política clara: 365 cópias diárias mais snapshots recentes, ou ajustar a promessa para 180 versões.
- Criar checksum e `schemaVersion` nos backups.
- Preservar o arquivo corrompido antes de qualquer recuperação.
- Validar importação por coleção antes de substituir dados.
- Transformar restauração em operação explícita: revisar, confirmar, publicar ou manter somente local.

### 3. Shows, séries e duplicação

**Problemas confirmados:**

- Parcelas previstas podem ser somadas como recebidas em alguns cálculos.
- NF mensal e NFs individuais não são mutuamente exclusivas.
- Emitir NF não define sempre `status: 'emitida'`.
- Marcar show recebido pode sobrescrever o histórico de parcelas.
- Editar série apaga e recria ocorrências futuras, podendo perder alterações manuais, NFs e recebimentos.
- Termo e prazo da série podem ficar inconsistentes.
- Exclusão de show/série pode deixar pagamentos e NFs órfãos.
- Duplicação de ocorrência de série mantém referências de série sem registrar uma exceção formal.

**Melhorias:**

- Regenerar séries por chave estável `serieId + data`, preservando IDs e históricos.
- Definir modo de faturamento: mensal agregado ou individual.
- Usar soft-delete ou exclusão com relatório de impacto.
- Preservar parcelas parciais ao marcar recebimento.
- Recalcular `recData`, prazo e referências ao duplicar show.

### 4. Equipe e pagamentos

**Problemas confirmados:**

- Existem duas versões de cálculo de saldo de integrante.
- Pró-labore aparece em alguns pontos, mas não entra de forma consistente no saldo.
- Pagamentos globais e pagamentos vinculados a show podem divergir.
- Um pagamento sem `showRef` não reduz necessariamente o pendente do show correto.
- Editar integrante pode alterar valores de shows passados, embora a mensagem indique shows futuros.
- Abrir e salvar show pode transformar escalação automática em manual.
- Ainda é possível cadastrar integrantes duplicados.
- Percentuais acima de 100% e valores negativos não são bloqueados.
- Histórico visual mostra somente os 30 pagamentos mais recentes.

**Melhorias:**

- Separar obrigação por show, pró-labore e pagamento avulso.
- Exigir vínculo do pagamento com show ou classificar explicitamente como avulso.
- Bloquear valores negativos e pagamentos acima do pendente, permitindo crédito excedente somente com confirmação.
- Preservar `manual` ao abrir e salvar um show.
- Atualizar apenas shows futuros ao alterar o valor padrão do integrante.
- Criar mesclagem de integrantes duplicados por nome normalizado, telefone e função.
- Adicionar filtros e paginação ao histórico.

### 5. Finanças e lançamentos

**Problemas críticos:**

- O lançamento por texto de receita pode ser gravado como despesa porque o tipo original é perdido.
- Valores com vírgula podem ser separados incorretamente antes do parser, como `99,90`.
- Parcelamento manual e parcelamento textual seguem regras diferentes.
- Parcelas podem iniciar no mês errado e perder centavos no arredondamento.
- Valores negativos são aceitos.
- Cartões não usam sempre a mesma competência de fatura no detalhe e no resumo.
- Configuração de fechamento/vencimento de cartões padrão pode não persistir.
- Imposto configurado para shows pode ser aplicado a receitas financeiras comuns.
- Existem fórmulas diferentes de caixa no dashboard e no painel financeiro.
- Categorias gravadas e categorias dos filtros não são totalmente compatíveis.

**Melhorias:**

- Criar parser monetário único para `99,90`, `1.500,00` e `R$ 5.800,00`.
- Preservar o tipo original de receita/despesa.
- Criar uma rotina única de parcelamento com ajuste final de centavos.
- Rejeitar valores menores ou iguais a zero quando não fizerem sentido.
- Separar competência, previsão e caixa realizado.
- Centralizar cálculo de imposto, categorias e cartões.

### 6. Relatórios e indicadores

**Problemas confirmados:**

- Relatório trimestral/anual pode considerar recebimentos de shows apenas do primeiro mês.
- Filtros de grupo/local não filtram todas as finanças e faturas.
- WhatsApp pode divergir do relatório visível.
- Previsão usa data do show em vez de vencimento/parcelas em vários trechos.
- Shows e NF mensal podem ser contabilizados duas vezes.
- Pagamentos de equipe no relatório não respeitam sempre o período selecionado.
- Categorias de saúde/casa/dívida usam regras diferentes.
- Datas com `toISOString()` podem deslocar o dia no fuso do Brasil.

**Melhorias:**

- Criar um único objeto de escopo do relatório: período, filtros e fonte de valor.
- Reutilizar o mesmo cálculo na tela, gráfico, exportação e WhatsApp.
- Usar recebimentos efetivos para caixa e vencimentos para previsão.
- Corrigir timezone com componentes locais.
- Mostrar claramente “competência”, “previsto” e “realizado”.

### 7. Notas fiscais e prazos

**Problemas confirmados:**

- NF emitida pode ser gravada sem status explícito.
- NF apenas emitida pode aparecer como recebida.
- Marcar NF mensal como recebida não registra data/valor efetivos de caixa.
- Shows de uma série continuam aparecendo como pendentes depois da NF mensal.
- Não há bloqueio de dupla emissão mensal + individual.
- `nfPrev` usa somente `saveLocal()`, sem sincronizar com JSONBin.
- Não há estado claro de atrasada, vence hoje ou recebida.

**Melhorias:**

- Criar schema único de NF.
- Separar emissão, lançamento no financeiro, vencimento e recebimento.
- Registrar `recebidaEm` e valor efetivo.
- Vincular NF aos shows cobertos.
- Adicionar estados: pendente, vence hoje, atrasada, emitida e recebida.
- Fazer `nfPrev` passar por `save()`.

### 8. Lançamento de show por texto

**Problemas confirmados:**

- Parser depende de rótulos rígidos.
- Formatos ISO podem ser interpretados incorretamente.
- Datas inválidas podem ser aceitas.
- Horário em formato “8 da noite” não é normalizado.
- Duração `1h30` não alimenta corretamente horas/minutos.
- Valor `R$ 5.800,00` não é extraído automaticamente.
- Nomes de integrantes no texto não são identificados.
- Toda a equipe pode ficar marcada por padrão.
- Reinterpretar o texto pode apagar ajustes feitos na prévia.
- Confirmação não exibe todos os campos que serão salvos.

**Melhorias:**

- Usar um objeto normalizado como fonte única da prévia e do salvamento.
- Extrair dinheiro, duração, datas, horários e aliases de rótulos.
- Bloquear confirmação com campos ausentes ou ambíguos.
- Mostrar equipe e valor de cada integrante na prévia.
- Preservar edição manual ao reinterpretar o texto.
- Mostrar o estado de sincronização depois de salvar.

### 9. Dashboard e calendário

**Problemas confirmados:**

- Alguns campos do dashboard entram em `innerHTML` sem escape.
- A tela pode parecer vazia durante carregamento lento.
- Hero e painel financeiro usam semânticas diferentes de saldo.
- Calendário e KPIs usam bases de data distintas.
- Filtros do calendário podem esconder shows ao trocar de mês.
- Estado vazio do calendário não é suficientemente claro.
- Navegação por “Mais” perde o estado ativo.
- No modo local, atualizar pode exibir “reenviando” mesmo sem configuração de nuvem.

**Melhorias:**

- Adicionar estados visuais loading, local, sincronizando, erro e vazio.
- Escapar todos os dados cadastrados.
- Unificar cálculos do hero e painel.
- Adicionar “Limpar filtros” no calendário.
- Mostrar “Nenhum show neste mês” ou “Nenhum resultado para este filtro”.

### 10. UX mobile, acessibilidade e PWA

**Problemas confirmados:**

- Zoom por pinça está desabilitado.
- Header e navegação podem continuar visíveis durante login/setup.
- Modais não têm gerenciamento completo de foco e Escape.
- Alguns alvos de toque são menores que 44px.
- Muitos cards clicáveis são `div` sem semântica de botão.
- Mensagens não usam `aria-live`/`role=alert`.
- Não há breakpoint abrangente para telas pequenas.
- `100vh` pode ser problemático no Safari com teclado/barras dinâmicas.
- Service worker só possui cache parcial e não cacheia respostas bem-sucedidas.

**Melhorias:**

- Permitir zoom.
- Usar `100dvh` e `env(safe-area-inset-bottom)`.
- Garantir alvos mínimos de 44px.
- Transformar cards clicáveis em botões acessíveis.
- Usar dialogs com foco, Escape e `aria-modal`.
- Adicionar `aria-live` aos estados de conexão e toast.
- Criar breakpoints para 320px, 375px e 414px.
- Atualizar a estratégia do service worker e versionar o cache.

---

## Proposta de layout moderno

A recomendação é uma modernização em três níveis, sem alterar a identidade Luah Produção.

### Direção visual

- Fundo escuro mais profundo, com superfícies elevadas e menos bordas.
- Dourado apenas para ações principais e destaques, não em todos os elementos.
- Verde/teal para conectado e recebido.
- Âmbar para pendente/atenção.
- Vermelho/rosa para erro, atraso e risco.
- Tipografia com hierarquia maior e menos texto em caixa alta.
- Cards mais compactos, com números grandes e descrições curtas.
- Mais espaço entre blocos e menos linhas decorativas.

### Nova página inicial

1. **Cabeçalho compacto**
   - Logo e nome.
   - Estado da conexão como chip clicável.
   - Botão de sincronização.
   - Avatar/conta ou menu de segurança.

2. **Card principal do mês**
   - “Caixa realizado” como número principal.
   - Comparação com previsão.
   - Data/hora da última atualização.
   - Ação “Ver detalhes”.

3. **Ações rápidas**
   - Novo show.
   - Lançar despesa.
   - Registrar pagamento.
   - Emitir/lançar NF.

4. **Calendário mensal**
   - Próximo ao topo.
   - Filtros em chips compactos.
   - Cores por status do show.
   - Mensagem de estado vazio.

5. **Resumo operacional**
   - Shows do mês.
   - Recebimentos pendentes.
   - Pagamentos da equipe.
   - Notas atrasadas.

6. **Atividade recente**
   - Últimos lançamentos.
   - Últimas sincronizações.
   - Últimos pagamentos.
   - Alertas de conflito ou backup.

### Nova navegação

- Início
- Shows
- Equipe
- Finanças
- Relatórios

O menu “Mais” deve ficar reservado para Backup, Conexão, Conteúdo, Equipamentos e configurações menos frequentes.

### Ordem recomendada de implementação visual

1. Corrigir segurança e fila de sincronização.
2. Corrigir fórmulas de caixa, recebimentos, NFs e equipe.
3. Criar componentes visuais consistentes: `Card`, `StatusChip`, `EmptyState`, `Modal`, `SectionHeader`.
4. Reorganizar dashboard e calendário.
5. Ajustar formulários para mobile.
6. Melhorar acessibilidade e PWA.
7. Fazer testes em Safari iOS e Chrome Android.

---

## Testes indispensáveis

### Sincronização

- Salvar offline e recarregar.
- Simular 401, 403, 404, 429, 5xx, CORS, DNS e timeout.
- Falhar PUT de exclusão e garantir que o registro não volte.
- Editar o mesmo show em dois aparelhos.
- Confirmar que apenas um polling fica ativo.

### Dados financeiros

- Valores `99,90`, `1.500,00` e `R$ 5.800,00`.
- Parcelamento 100/3 com soma exata.
- Pagamento acima do saldo.
- Recebimento parcial e depois total.
- NF mensal versus individual.
- Relatório mensal, trimestral e anual.

### Mobile

- Safari iOS com teclado aberto.
- Android com tela de 320px.
- Zoom em 200% e 400%.
- Navegação por teclado.
- Fechamento de modal com Escape.
- Instalação PWA e atualização do cache.

---

## Perguntas de produto que precisam ser definidas

1. A fonte oficial dos dados será JSONBin, um backend próprio ou somente o aparelho?
2. A API Key master poderá ser revogada e substituída por uma arquitetura segura?
3. O backup desejado é de 365 dias, 180 versões ou snapshots por alteração?
4. Ao restaurar, a nuvem deve ser substituída, mesclada ou apenas o aparelho recuperado?
5. O relatório principal deve mostrar caixa realizado, competência ou ambos separados?
6. NF mensal e NF individual serão alternativas exclusivas?
7. Como tratar pró-labore, pagamentos avulsos e pagamentos acima do previsto?
8. O redesign deve priorizar uso no celular vertical ou também tablets/desktop?

## Conclusão

A prioridade correta não é começar apenas pelo visual. Primeiro é necessário proteger a fila, esclarecer os erros de sincronização e corrigir as fórmulas financeiras. Depois disso, o redesign pode ser implementado com segurança sobre uma base consistente.

A modernização visual recomendada é viável e deve resultar em um app mais limpo, rápido de entender e adequado ao uso diário no celular, mas precisa ser acompanhada de uma camada de dados mais confiável para que uma interface bonita não esconda perdas ou divergências financeiras.
