# Composição de Caixa — Carneiro Ganhador Loterias

Aplicação interna de web para controle de caixa da Carneiro Ganhador Loterias: lançamentos diários (saídas e débitos), fechamento de caixa por terminal, ficha de clientes fiado, dashboard consolidado e área administrativa/gerencial.

## Como funciona

O app é um único arquivo estático ([`index.html`](index.html)) — HTML, CSS e JavaScript no mesmo arquivo, sem build ou dependências instaláveis. Os dados são armazenados de forma compartilhada e sincronizados em tempo real via [Supabase](https://supabase.com/) (tabela `app_storage`).

Bibliotecas carregadas via CDN:
- [SheetJS (xlsx.js)](https://github.com/SheetJS/sheetjs) — exportação de relatórios para Excel.
- [supabase-js](https://github.com/supabase/supabase-js) — cliente do Supabase.

## Funcionalidades principais

- **Login** por funcionário(a), com cadastro de novos usuários e recuperação de senha por pergunta de segurança.
- **Terminal do dia**: seleção obrigatória do terminal operado a cada dia.
- **Lançamentos**: registro de saídas e débitos por tipo de atividade (Fiado, Pix, Cheque, Rede, Troca de caixa, etc.), com histórico filtrável.
- **Fechamento de caixa**: composição diária por terminal (troco, produtos, bolão, relatório do terminal) com cálculo automático dos Totais 1, 2 e 3.
- **Ficha de clientes / Fiado**: cadastro de clientes, pagamentos de fiado e acompanhamento de saldo devedor.
- **Histórico, Geral e Dashboard**: consultas consolidadas por período, terminal, funcionária e tipo de atividade.
- **Gerência e Admin**: estoque gerencial, troco do cofre, batimento gerencial, redefinição de senha de funcionários e exportação para Excel.

## Uso

Basta abrir o [`index.html`](index.html) em um navegador — não há etapa de instalação ou build. A conexão com o Supabase já vem configurada no próprio arquivo.

No primeiro acesso (sem nenhum usuário cadastrado ainda), o sistema cria automaticamente um usuário administrador padrão para o primeiro login; troque a senha dele assim que possível pela aba **Admin**.
