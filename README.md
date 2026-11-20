# Composição de Caixa — Carneiro Ganhador Loterias

Aplicativo web de **arquivo único** (`composicao_caixa_carneiro_ganhador.html`, HTML + CSS + JS puro) para controle de caixa, fechamento, fiado, estoque/troco da gerência e relatórios de sócios de uma lotérica.

- **Dados:** Supabase, tabela `app_storage` (`key` TEXT PK, `value` JSONB array, `updated_at`).
- **Publicação:** GitHub Pages. Depois de publicar, todos precisam atualizar com **Ctrl+F5**.
- **Sincronização:** o app recarrega os dados a cada 5 s (`atualizarTudo`).
- Guia técnico para quem altera o código: `CLAUDE.md`.

---

## 1. Perfis de acesso

Definidos em **Admin → Gerenciar funcionário → Perfil de acesso** (campo `perfil` do usuário). Cada perfil superior **inclui tudo** do anterior.

| Perfil | O que acessa |
|---|---|
| **Caixa (normal)** | Lançamentos, Fechamento, Clientes/Fiado, Histórico e Geral dos seus caixas |
| **Gerência** | Caixa + Composição, Envios, Saídas e histórico completo (todos os usuários) |
| **Administrador** | Gerência + Admin, Gestão de Fiado, Dashboard, backup, terminais, usuários |
| **Sócio** | Tudo (Caixa + Gerência + Administrador) + módulo **Sócio** (botão 🤝 no menu, com "← Voltar ao app") |

Regras de proteção: o usuário `admin` é sempre Administrador; ninguém retira o próprio acesso de Administrador; sempre existe ao menos um Administrador ativo. A mudança vale no próximo login da pessoa. Usuários antigos sem `perfil` migram sozinhos: sócio → Sócio; gerência ou admin → Administrador.

Outras ações em Gerenciar funcionário: redefinir senha, bloquear/desbloquear, apagar usuário, e tabela com o perfil de todos.

## 2. Login e terminais

- Login por usuário e senha; recuperação por pergunta de segurança; cadastro de novos usuários.
- Usuário bloqueado não entra.
- Cada funcionária escolhe o **terminal do dia** ao entrar (pré-preenche o último usado). O Administrador pode entrar sem terminal cadastrado, para poder cadastrá-los.
- Terminais são cadastrados em Admin → Gestão de Terminais (sem números fixos no código). Tipos: **Terminal caixa**, **Terminal gerência**, **Terminal bilhetes**. Admin também verifica terminais do dia.

## 3. Lançamentos (aba inicial)

Novo lançamento de **Saída** ou **Entrada (Troco)**, com data (padrão hoje, sem futuro), valor, observação e terminal. Rótulo na tela: *Troco* = **Entrada** (valor negativo no cálculo); *Saida* = **Saída** (positivo).

Atividades de Saída (caixa): Brindes/Descontos, Cheque, Conta Loja (com detalhe), Depósito 003, Envio cofre antes/depois do carro forte, Fiado, Market Place, Notas (exige categoria: Escritório, Limpeza, Cozinha, Manutenção, Outros), **Pix anterior**, Pix fora terminal, Prêmios, Raspadinha premiada, Rede, Bolão, **Bolão Especial**, Troca de caixa, Troca Telesena, Outros.
Atividades no terminal de **Bilhetes**: Envio dinheiro, Fiado, Brindes/Descontos, Pix fora terminal, Rede, Devolução bilhetes, Notas.

Comportamentos especiais:
- **Fiado:** pede o cliente; lança saída com `clienteId`.
- **Troca de caixa / Envio dinheiro:** lançamento duplo (Saída no terminal de origem, Entrada no destino).
- **Pix anterior:** Pix recebido referente a outro dia; campo **Data do Pix (opcional)**; se preenchida, vai no lançamento (`dataPix`) e na observação como `[Pix de dd/mm/aaaa]`; não pode ser posterior à data do lançamento.
- **Bolão Especial:** só controle do caixa; entra na saída/fechamento do caixa e fica fora de Batimento Gerencial, aba Saídas e relatório dos sócios.
- Lista de lançamentos com excluir/restaurar (exclusão lógica) e **pagamentos de fiado** na mesma lista (badge "Pgto fiado"), também com excluir/restaurar. Créditos `CARGA30-P-*` são protegidos.

## 4. Fechamento de caixa

Por terminal e data. Campos: **c** Troco terminal, **d** Composição de produtos (quantidade × valor), **e** Bolão, **f** valor do relatório do terminal.
- `Total2 = (a − b) + c + d + e` (a = saídas, b = entradas)
- `Total3 (Batimento) = (Total2 − Total1) − f`
- **Total1 = Total2 do último dia contábil com dados** anterior à data (a loja não abre domingos/feriados). Todas as telas mostram a data de referência usada.
- Fechamentos registrados listados/consultados; admin vê todos, com filtros e exportação para Excel.

### Fechamento de bilhetes
- **k** Vitrine de Bilhetes (quantidade × valor por item), **l** Gaveta bilhete, **j** saídas do terminal de bilhetes.
- `Total bilhete = j + k + l`
- **Batimento bilhete = (k + l do último dia contábil do terminal) − Total bilhete d0.** A tela informa a data do último dia contábil considerado. Gravado no fechamento (`batimentoBilhete`, `dataRefBatimento`, `vitrineGavetaAnterior`).

## 5. Clientes e Fiado

- **Ficha de clientes:** cadastro (nome, celular, obs), código `CLI-nnnn` gerado como **maior código usado + 1** (nunca reutiliza), busca, saldo devedor por cliente.
- **Saldo = Σ fiado (não excluído, não `legado`) − Σ pagamentos válidos.**
- **Pagamento de Fiado:** valor, cliente, observação; aparece na lista de lançamentos; exclusão lógica.
- **Gestão de Fiado (Admin/Gerência+):** saldo por cliente, fiado por dia, linha do tempo, pagamentos, recebimentos.
- Carga histórica (30/10/2026): registros `CARGA30-F-*` / `CARGA30-P-*`, caixa `0000`, usuário `sistema`.
- Fiado **não entra** nas Saídas do Batimento.

## 6. Histórico e Geral

- **Histórico de registros:** lançamentos do usuário (Gerência/Admin veem todos, com filtro extra de funcionária).
- **Geral:** totais gerais, lançamentos por dia, por terminal, por funcionária e por tipo de atividade.

## 7. Composição (Gerência)

- **Data única sincronizada** entre Composição, Entrada de produtos, Envio de troco e Envio de produtos; aviso "⚠ Lançando em dia anterior".
- **Estoque de produtos gerencial**, **Vitrine raspadinha**, **Troco cofre** (por denominação), **Caixa gerencial** (c+d+e de todos os terminais), **Bilhete gerencial**, **Bolões especiais**, **Gaveta Gerencial**, **Fiado total**.
- **Bilhete gerencial = Vitrine de bilhetes (k) + Gaveta bilhete (l)** do último fechamento de cada terminal de bilhetes na data. (Antes só a gaveta entrava.)
- `TOTAL2 gerencial = Estoque + Vitrine raspadinha + Troco cofre + Caixa gerencial + Bilhete gerencial + Bolões especiais + Gaveta + Fiado total`.
- Composição gravada por dia (`total2Gerencial`); serve de **TOTAL1** do dia seguinte.

### Batimento Gerencial
- **Débito = TOTAL1 + Total Relatórios (f) + Entrada Produtos − devolução**
- **Crédito = TOTAL2 gerencial + Saídas hoje**
- **Diferença = Crédito − Débito**
- **Saídas hoje** (critério único, igual ao da aba Saídas): saídas dos terminais caixa e gerência + **Brindes/Descontos, Pix fora terminal e Rede do caixa Bilhetes**; exceto Fiado, Bolão, Bolão Especial, Troca de caixa, Devolução bilhetes e Troco cofre. Tabela por atividade com quantidade clicável (abre os lançamentos).
- **Entrada Produtos − devolução** = entradas de produtos da data − lançamentos "Devolução bilhetes" do caixa Bilhetes.
- Fórmulas visíveis na tela ("Cálculo: …") para conferência.

## 8. Envios

- **Envio de troco do cofre:** Saída (+) na gerência ("Troco enviado"), Entrada (−) no terminal que recebeu; abate do Troco cofre.
- **Envio de produtos:** Entrada (−) na origem e no destino; abate o Estoque gerencial.
- **Entrada de produtos (compra):** lista de produtos, quantidades, valor; **Alterar/Excluir** (exclusão lógica, reversão do estoque, histórico de edições).

## 9. Saídas (Gerência)

Filtros: data inicial/final, terminal, funcionária. Por padrão "Caixas + Gerência (+ Brindes, Pix e Rede do Bilhetes)". Tabela por atividade (A–Z) com valor e quantidade clicável. Mesmo critério de saída do Batimento Gerencial.

## 10. Dashboard e Admin

- **Dashboard:** visão consolidada de saídas e entradas de todos os perfis, filtros por período e terminal, análise detalhada, consulta de vendas por dia, fechamento semanal/mensal de vendas.
- **Admin:** consulta por período (Total1/Total2/Total3, exportação Excel), fechamentos registrados, Gestão de Terminais, Gerenciar funcionário (perfil, senha, bloqueio), **Backup e Restauração** (cópia criptografada AES-GCM protegida por senha, lembrete diário), **Integração com Excel (Office 365)** via webhook do Power Automate.

## 11. Módulo Sócio

Upload do relatório do dia (foto/PDF, leitura por OCR com pdf.js), identificação (terminal e data), consulta por data, checklist de terminais. Vendas = (saldo inicial + entradas) − saldo final. Usuários **Sócio** abrem pelo menu 🤝 e voltam com "← Voltar ao app".

## 12. Regras de armazenamento e segurança

- Cada `value` é um array JSONB cru; nunca `JSON.stringify` dentro do JSONB.
- Não grava se a leitura falhou (`LEITURA_OK`); cache de 4 s com invalidação após cada salvar.
- Exclusões são lógicas (`excluido:true`).
- Textos de dados passam por `escaparHtml`.
- Campos só numéricos usam `inputmode` (`decimal`, `numeric`, `tel`) para abrir o teclado numérico no celular.

## 13. Ferramentas de manutenção (HTML separados, rodar no navegador)

Todas seguem: backup antes, releitura antes de gravar, verificação depois, botão de restaurar, confirmação digitada.
`corrigir_codigos_carga30out.html`, `auditoria_carga_30out_por_nome.html` (somente leitura), `comparacao_fiado_sistema_x_excel_0411.html`, `zerar_fiados_e_carga_30out.html`, `unificar_clientes_fiado.html`, `varredura_lancamentos_gerencia.html`, `fiado_app_x_excel_0411.html`.

## 14. Histórico de mudanças recentes

- Perfis de acesso por usuário; Administrador inclui Gerência; Sócio com acesso total.
- Saídas da Composição (Batimento) alinhadas com a aba Saídas (inclui Brindes/Pix/Rede do Bilhetes).
- Bilhete gerencial passa a somar Vitrine + Gaveta.
- Novo lançamento **Pix anterior** com data opcional.
- Batimento do fechamento de bilhetes com data do último dia contábil.
- Pagamentos de fiado na lista de lançamentos; Bolão Especial; teclado numérico.
- Geração de código de cliente sem reutilização.

> **Atenção ao publicar:** composições gerenciais **já gravadas** antes desta versão têm o Bilhete gerencial só com a gaveta. O TOTAL1 do primeiro dia depois da mudança pode ficar diferente pelo valor da vitrine de bilhetes daquele dia anterior.
