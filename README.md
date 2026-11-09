# CLAUDE.md — Sistema de Composição de Caixa · Carneiro Ganhador Loterias

Guia de aprendizado do sistema. Leia antes de qualquer alteração. Idioma de trabalho: **português do Brasil**; estilo: conciso, caloroso, sênior.

## 1. Visão geral
- App web de **arquivo único** (`composicao_caixa_carneiro_ganhador.html`, HTML + CSS + JS puro) para controle de caixa de uma lotérica (terminais, fechamento, fiado, estoque e troco da gerência, relatórios de sócios).
- Dados no **Supabase**, tabela `app_storage` (`key` TEXT PK, `value` JSONB, `updated_at`). Publicação no **GitHub Pages**. Depois de publicar, os usuários precisam atualizar com **Ctrl+F5**.
- O sandbox de desenvolvimento **não alcança o Supabase (403)**. Ferramentas de manutenção são arquivos HTML que a usuária abre no navegador (URL `https://psebppouysydhyvuujss.supabase.co`, chave publicável no próprio arquivo).

## 2. Armazenamento (regras que não podem ser quebradas)
- Cada `value` é um **array JSONB**. Sempre gravar o array cru (upsert), **nunca** `JSON.stringify` dentro do JSONB.
- No app: `window.storage.get/set` (o `set` recebe string JSON e faz parse), guarda `LEITURA_OK` (não gravar se a leitura falhou), `carregarComCache` (TTL 4 s) e `invalidarCache(chave)` após todo salvar.
- Chaves principais: `registros_loterica`, `fechamentos`, `clientes_loterica`, `pagamentos_fiado_loterica`, `usuarios_loterica`, `terminais_loterica`, composição da gerência, `entradas_produtos_gerencia_loterica`, atribuição terminal×dia.
- **Padrão de ferramentas de manutenção:** backup baixado antes, reler antes de escrever, gravar arrays, reler e verificar depois, botão de restaurar, confirmação digitada, nunca apagar sem backup.

## 3. Modelo de dados
- **Registro (lançamento):** `{id, usuario, nome, data, hora, caixa, tipo:'Saida'|'Troco', atividade, clienteId, clienteNome, valor, obs, excluido, timestamp}`; transferências ganham `parId`, `registradoPor`.
- **Rótulos da UI:** `Troco` aparece como **Entrada** (valor negativo no cálculo); `Saida` como Saída (positivo). Valores internos de `tipo` não mudam.
- **Fiado:** `atividade:'Fiado'`, `clienteId` `CLI-nnnn`. Saldo = Σ Fiado (não excluído e não `legado`) − Σ pagamentos (`calcularSaldosClientes`). O flag `legado:true` faz o registro ser ignorado (5 filtros usam `!r.excluido && !r.legado`). Fiado **não entra** nas Saídas do Batimento Gerencial.
- **Pagamento de fiado:** `{id, clienteId, clienteNome, valor, obs, usuario, nome, data, hora, timestamp, excluido?, excluidoPor?, excluidoEm?}`. Aparece na lista de lançamentos (aba Histórico) com badge "Pgto fiado" e botão excluir/restaurar (exclusão lógica, `excluido:true`); **não entra** nos totais de Saída/Entrada do caixa. `carregarPagamentosFiado()` devolve só os válidos (use em saldos/relatórios); `carregarPagamentosFiadoBruto()` devolve todos (**use só para gravar**, senão os excluídos são apagados de vez). Créditos `CARGA30-P-*` não podem ser excluídos pela lista. Ferramentas que somam pagamentos devem ignorar `excluido`.
- **Entrada de produtos:** `{id, usuario, nome, data, hora, produtos:[{id,nome,valorUnit,qtd,subtotal}], total, timestamp, excluido?, historicoEdicoes?}`. Lista de produtos em `PRODUTOS_ESTOQUE_GERENCIA`.
- **Atividades (constantes no código):** `ATIVIDADES_SAIDA_BILHETES` (inclui Envio dinheiro, Fiado, Brindes/Descontos, Pix fora terminal, Rede, Devolução bilhetes, Notas), `ATIVIDADES_TRANSFERENCIA_TERMINAL` (Troca de caixa, Envio dinheiro), `ATIVIDADES_FORA_DEBITO_GERENCIAL` (Bolão, **Bolão Especial**, Troca de caixa, Devolução bilhetes; Bolão Especial = só controle do caixa, entra na Saída/fechamento do caixa e fica fora de Batimento, aba Saídas e relatório dos sócios). `Notas` exige categoria (Escritório, Limpeza, Cozinha, Manutenção, Outros), gravada na `obs` como `[Categoria] texto`.

## 4. Perfis e terminais
- Perfis: funcionária de caixa, **Gerência**, **Admin**, **Sócio** (módulo isolado, sem terminal, upload de relatório em foto/PDF via pdf.js).
- Terminais são cadastrados no Admin, **sem números fixos no código**. Tipos: Terminal caixa, Terminal gerência, Terminal bilhetes. Toda agregação filtra por tipo (`numerosTerminaisCaixa()`, `terminaisPorTipo()`).
- Quem vê o quê é decidido no login (`EH_ADMIN`, `EH_GERENCIA`); os botões do menu são ocultados/exibidos no início da inicialização para evitar "menu sumido". Menu Saídas e Composição: só Gerência.

## 5. Regras de sinal e cálculos
- **Troca de caixa:** Saída (+) na origem, Entrada (−) no destino.
- **Troco (suprimento):** Entrada (−) nos dois terminais.
- **Troco cofre:** Saída (+) na gerência ("Troco enviado"), Entrada (−) no terminal que recebeu ("Troco recebido").
- **Envio de produtos:** Entrada (−) na origem e no destino; abate o Estoque de produtos gerencial.
- **Fechamento de caixa:** `Total2 = (a−b) + c + d + e`; `Total3 = (Total2 − Total1) − f`. `a` = Saídas, `b` = Entradas (Troco), `c` troco, `d` composição, `e` bolão, `f` relatório.
- **Total1 / "dia anterior"** = **último dia contábil com dados registrados** anterior à data (a loja não abre aos domingos e em alguns feriados). Helper `ultimaDataComFechamento(fechamentos, terminais, dataRef)`. Todas as telas (Fechamento, Batimento Gerencial, Painel Geral, Admin) **mostram a data de referência usada** ("ref. dd/mm"). Nunca usar `diaAnteriorISO` literal em cálculo contábil.
- **Batimento Gerencial:** Débito = TOTAL1 + Total Relatórios + **Entrada Produtos − devolução**; Crédito = TOTAL2 + Saídas hoje (terminais caixa, exceto Fiado, Bolão, Troca de caixa e Devolução bilhetes). Diferença = Crédito − Débito.
- **Entrada Produtos − devolução** = Σ entradas de produtos da data (não excluídas) − Σ lançamentos "Devolução bilhetes" (Saída) do caixa Bilhetes na data. A devolução **vem do caixa Bilhetes**, não de um formulário próprio.
- **Estoque/troco da gerência:** o saldo inicial de um dia vem da composição mais recente anterior (`obterSaldoProdutosAntesDe`). Relatórios de sócios: vendas = (saldo inicial + entradas) − saldo final.

## 6. Datas — lição importante
- Transações antigas ficaram erradas porque envio de produtos/troco assumia "hoje". Regra: **toda transação tem seletor de data, padrão hoje**, permitindo dias anteriores.
- Lançamentos, pagamento de fiado e Fechamento têm campo próprio. Na Gerência (Composição, Entrada de produtos, Envio de troco, Envio de produtos) a data é **uma só e sincronizada** (`mudarDataGerencia`): mudar um campo atualiza os outros e recarrega a composição daquele dia; aviso "⚠ Lançando em dia anterior" quando ≠ hoje; sem datas futuras.
- Abater um dia passado **não altera automaticamente** composições de dias posteriores já registradas.

## 7. Funcionalidades recentes (histórico)
- Carga FULL de fiados (08/10/26): total R$ 87.603,02, clientes em MAIÚSCULAS, descrição "Carga sistêmica - Saldo inicial 08/10/26", exatamente como a aba GERAL (inclui valores fixos digitados; linhas sem nome não entram). Banco foi zerado de propósito. Fiados de 08/10 foram movidos para 07/10 por ferramenta própria.
- Entrada de produtos com **Alterar/Excluir** (exclusão = `excluido:true`, reversão do estoque, histórico de edições). Menu **Saídas** (filtros de data inicial/final, terminal, funcionária; tabela com grade e fonte reduzida).

## 8. Armadilhas conhecidas (e como foram resolvidas)
- **`confirm()` nativo pode ser bloqueado** pelo navegador/ambiente e o botão "não faz nada". Usar confirmação inline na própria tela.
- **A sincronização a cada 5 s (`atualizarTudo`) redesenha listas** e apaga o que a pessoa está digitando. Listas com editor aberto não podem ser re-renderizadas (guarda `EDITOR_RENDERIZADO_ID`).
- **Bloqueios silenciosos** (ex.: estoque ficaria negativo) deixam a pessoa sem saber o que houve. Preferir permitir e **avisar** em vermelho, com mensagem que não desaparece depressa.
- Células Excel com texto iniciando em "=" viram fórmula (#VALUE!). Prefixar texto ao gerar planilhas.
- Ferramenta de carga que bloqueia quando a lista vem vazia: tratar `null`/vazio como `[]` e avisar, pois banco zerado pode ser intencional.
- Valores monetários negativos: `formatarMoeda` já usa `toLocaleString`.
- Todo texto vindo de dados deve passar por `escaparHtml` antes de entrar em `innerHTML`.
- **Recarga de fiados reaproveita CLI-0001…**: após zerar e recarregar, o mesmo código passa a ser outra pessoa. O combo de clientes só é redesenhado em `popularSelectClientes`; uma aba aberta antes da carga mostra o nome antigo e grava no código atual (caso Seiji → ZENILDO, R$ 100,00). O `clienteNome` é buscado no cadastro **no salvamento**, então não revela o erro: só a comparação com o Excel por nome revela. Depois de recarga, todos precisam Ctrl+F5; **Corrigido**: o app agora gera o código como maior número usado + 1 (nunca `clientes.length+1`), e `corrigir_codigos_carga30out.html` troca o código do cadastro da carga repetido (só mexe em CARGA30-*; lançamentos de pessoas ficam intocados).
- Auditoria de carga: sempre por **nome**, nunca por linha do Excel ou código antigo (`auditoria_carga_30out_por_nome.html`, somente leitura).

## 9. Como trabalhar neste projeto
1. Editar o HTML por substituição exata de texto (script Python com `assert count == 1`), nunca reescrever o arquivo inteiro.
2. Após editar: extrair o maior `<script>` e rodar `node --check`. Sem acesso ao Supabase, validar a lógica por leitura e, quando possível, com dados simulados.
3. Entregar o arquivo atualizado e lembrar: **publicar no GitHub Pages e Ctrl+F5**.
4. Mudanças em dados reais: sempre ferramenta HTML separada, com backup e verificação (seção 2).
5. Antes de mexer em cálculo contábil, conferir a seção 5 e manter fórmulas visíveis na tela ("Cálculo: ...") para a usuária auditar.
6. Em dúvida sobre regra de negócio, perguntar uma vez, de forma objetiva; pedidos claros executar direto.

## 10. Pendências conhecidas
- A usuária deve rodar `carga_full_fiados.html` (depois de publicar o app com o flag `legado`) e conferir se o total fecha em R$ 87.603,02.
- Lançamentos antigos de envio de produtos/troco gravados com data errada ainda não foram corrigidos (possível ferramenta de correção de data em massa).
- Auditar se relatórios de sócios devem tratar produtos enviados a outro terminal como saída de estoque distinta de venda.
