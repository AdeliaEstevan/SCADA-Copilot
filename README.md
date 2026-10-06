# SCADA COPILOT · Fase 1 (MVP)

Assistente técnico da Fábrica SCADA que responde com base nos procedimentos **vigentes**, sempre com documento, versão e página. Esta é a primeira versão real (com banco de dados) do protótipo apresentado à gestão.

Versão: **2026.10.05A**

---

## 1. Arquivos

| Arquivo | Para que serve |
|---|---|
| `index.html` | O sistema inteiro, em um arquivo só (HTML + CSS + JS). Publica no GitHub Pages. |
| `supabase/01_schema.sql` | Cria tabelas, regras de acesso (RLS), funções de curadoria, auditoria e o bucket dos PDFs. |
| `supabase/02_demo_opcional.sql` | Opcional. Carrega os 13 documentos fictícios da demonstração, marcados como `demo`. |

## 2. O que a Fase 1 entrega

- Login com perfis **Colaborador**, **Gestão** e **Administrador**, primeiro acesso com criação de senha e termo de uso dos dados (LGPD).
- **Início** com indicadores: consultas, taxa de resposta, perguntas sem resposta, avaliações úteis, documentos vigentes e evolução semanal (semana de domingo a sábado).
- **Assistente Técnico**: pergunta em linguagem natural, pedido de complemento quando falta o equipamento, resposta com passos, cuidados e cards de fonte (documento, versão, página, trecho), alerta de atividade crítica, avaliação útil/não útil com motivo e painel de rastreabilidade.
- **Base de Conhecimento**: lista com filtros, histórico de versões, envio de PDF/TXT com extração de texto por página, divisão em trechos, revisão e homologação.
- **Histórico de Consultas**: cada pessoa vê as suas; Gestão e Administrador veem as da equipe.
- **Lacunas de Conhecimento**: perguntas sem orientação homologada e relatos da equipe, agrupados, com status.
- **Administração**: perfis de acesso, ativação/desativação, auditoria somente-inserção e regras do assistente.

Fica para a **Fase 2**: incidentes, lições aprendidas, inteligência operacional, alertas, treinamentos e procedimentos críticos (aparecem no menu como "Fase 2").
Fica para a **Fase 3**: login corporativo (SSO), integrações e notificações.

## 3. Testar sem banco (modo demonstração)

Abra o `index.html` no navegador sem preencher a configuração. Ele entra em **modo demonstração**: escolha um perfil, os dados são fictícios e somem ao recarregar. Serve para treinar a equipe e apresentar o fluxo.

Roteiro rápido: entre como Administrador › Base de Conhecimento › Enviar e processar › "Usar documento de exemplo (POP-033)" › Processar › Revisar trechos e homologar › pergunte no assistente "Existe POP para teste de telecomando em comissionamento?".

## 4. Implantação

### 4.1 Criar um projeto Supabase novo
1. Em supabase.com, crie um projeto **exclusivo do Copilot** (sugestão de nome: `copilot-fabrica-scada`, região São Paulo).
2. **Anote o ID do projeto** (o trecho antes de `.supabase.co`). Ele é diferente dos projetos do Sobreaviso e do Chronos — confira antes de rodar qualquer SQL.

### 4.2 Criar o banco
1. Abra **SQL Editor › New query**, cole todo o `supabase/01_schema.sql` e clique em **Run**.
2. (Opcional, para testar com conteúdo) Rode também `supabase/02_demo_opcional.sql`.

### 4.3 Configurar a autenticação

_Os nomes dos menus do Supabase mudam de tempos em tempos; se algum não bater, procure pelo nome da opção._
Em **Authentication**:
- **Sign In / Providers › Email**: deixe o login por e-mail ligado e **desligue "Allow new users to sign up"**. Só a curadoria cria usuários.
- **URL Configuration**: em **Site URL** e em **Redirect URLs**, coloque o endereço onde o sistema vai ficar (ex.: `https://SEU-USUARIO.github.io/scada-copilot/`). É para lá que os links de convite e de redefinição de senha levam.

### 4.4 Criar a primeira administradora
1. **Authentication › Users › Add user**:
   - *Send invitation*: a pessoa recebe um link, cria a senha e entra; ou
   - *Create new user* com senha provisória e **Auto Confirm User** marcado (use este se o e-mail de convite não chegar — veja a observação abaixo). Depois do primeiro login, a pessoa troca a senha pelo cadeado no rodapé do menu.
2. No **SQL Editor**, promova essa pessoa a administradora:
   ```sql
   update public.perfis
      set papel = 'admin', nome = 'Nome Completo', matricula = 'D-0000', funcao = 'Curadoria da base'
    where id = (select id from auth.users where email = 'email.da.pessoa@empresa.com');
   ```
   A partir daí, perfis e dados das demais pessoas são ajustados pela tela **Administração**.

> **E-mails de convite:** o servidor de e-mail padrão do Supabase tem limite baixo de envio e pode não entregar para endereços fora da equipe do projeto. Se os convites não chegarem, use *Create new user* com senha provisória ou configure um SMTP corporativo em **Project Settings › Authentication › SMTP Settings**.

### 4.5 Ligar o `index.html` ao projeto
1. Em **Project Settings › API**, copie a **Project URL** e a chave **anon public**.
2. No `index.html`, procure o bloco `CONFIG` (logo no início do script) e preencha:
   ```js
   SUPABASE_URL: 'https://SEU-PROJETO.supabase.co',
   SUPABASE_ANON_KEY: 'eyJ...',
   ```
3. Ajuste, se quiser, a lista `EQUIPAMENTOS` (nomes e sinônimos que o assistente reconhece nas perguntas).

> A chave **anon** é pública por natureza: quem protege os dados são as regras de acesso do banco. **Nunca** coloque a chave `service_role` no arquivo.

### 4.6 Publicar no GitHub Pages
1. Crie um repositório (ex.: `scada-copilot`) e envie o `index.html` para a raiz.
2. **Settings › Pages › Build and deployment**: *Deploy from a branch*, branch `main`, pasta `/ (root)`.
3. Acesse o endereço gerado e confirme que é o mesmo cadastrado no item 4.3.

O arquivo carrega duas bibliotecas de CDN: `cdn.jsdelivr.net` (cliente Supabase) e, só no envio de PDF, `cdnjs.cloudflare.com` (leitor de PDF). Se a rede da empresa bloquear algum desses endereços, peça a liberação à TI.

## 5. Rotina da curadoria

1. **Base de Conhecimento › Enviar e processar**: arraste o PDF. O código é lido do nome do arquivo quando possível (ex.: `POP-018_recuperacao_utr.pdf`).
2. Confira os dados, escreva o que mudou e clique em **Processar documento**. A versão é gravada **em revisão** e ainda não é fonte do assistente.
3. Clique em **Revisar trechos e homologar**. Confira se seções, passos e cuidados foram separados corretamente.
4. **Homologar**: a nova versão vira a única vigente; a anterior vira obsoleta automaticamente, na mesma operação.
5. Para retirar um documento de uso sem substituto: abra a versão vigente › **Tornar obsoleta**. Para desistir de uma versão em revisão: **Descartar versão**.

**Como preparar um bom PDF:**
- Texto selecionável (PDF gerado a partir do Word, não escaneado). Páginas digitalizadas aparecem como aviso e precisam de OCR antes do envio.
- Títulos de seção numerados e curtos (`3. Recuperação do canal`).
- Passos numerados (`1)`, `2)`… ou `1.`, `2.`…), um por linha.
- Uma seção com "Cuidados", "Restrições" ou "Pré-condições" no título — ela vira o bloco de cuidados das respostas.
- DOCX: salve como PDF antes de enviar.

## 6. Carregar os primeiros POPs reais

1. Escolha os **10 a 15 POPs mais consultados** pela equipe.
2. Envie e homologue cada um seguindo a seção 5.
3. Se rodou o seed de demonstração, apague os fictícios:
   ```sql
   delete from public.documentos where demo;
   ```
4. Faça 20 perguntas reais com 2 ou 3 colegas e confira as fontes antes de liberar para todos.

## 7. Segurança garantida pelo banco

Estas regras valem mesmo que alguém tente usar a API diretamente, sem a tela:

- Só existe **uma versão vigente** por documento (índice único).
- Colaborador e Gestão **não recebem o conteúdo** de versões obsoletas ou em revisão, nem o PDF delas.
- Só Administrador cria documentos, envia versões, homologa e torna obsoleto; homologação e obsolescência passam por funções que verificam o perfil.
- Ninguém altera o próprio perfil; o sistema impede remover o **último administrador ativo**.
- Cada pessoa registra e vê as próprias consultas; Gestão e Administrador veem as da equipe.
- A **auditoria** é somente-inserção: nem administradores editam ou apagam registros. Mudanças de versão e de perfil são auditadas automaticamente.

**Testes executados nesta versão:**
- 26 testes das regras de acesso em PostgreSQL (permissões por perfil, homologação atômica, versão única vigente, auditoria imutável, PDF restrito à versão vigente).
- Teste ponta a ponta com o cliente oficial do Supabase sobre PostgREST: login, envio de TXT, criação de trechos, homologação, consulta como colaborador usando o novo documento, avaliação, lacuna automática, histórico e indicadores.
- Navegação completa no modo demonstração em desktop e celular, temas claro e escuro.

## 8. Limitações conhecidas da Fase 1

- **Busca lexical** em português com sinônimos do domínio. Perguntas com vocabulário muito diferente do texto do POP podem não encontrar a resposta — nesse caso o assistente diz que não encontrou e a pergunta vira lacuna. A busca semântica (embeddings) pode entrar quando o provedor for definido com a TI.
- A base vigente é carregada no navegador ao entrar. Funciona bem até alguns milhares de trechos; acima disso, a busca deve ir para o servidor.
- Sem IA generativa: as respostas são montadas diretamente dos trechos oficiais.
- Leitura de DOCX e OCR de páginas escaneadas não estão incluídos.

## 9. Como avaliar o piloto (30 dias)

| Indicador | Onde ver | Meta sugerida |
|---|---|---|
| Taxa de resposta | Início | acima de 80% |
| Respostas avaliadas como úteis | Início / Histórico | acima de 75% |
| Lacunas abertas | Lacunas de Conhecimento | caindo semana a semana |
| Consultas por semana | Início (gráfico) | uso recorrente da equipe |

Com esses números, a proposta da Fase 2 chega à gestão com dados reais de uso.
