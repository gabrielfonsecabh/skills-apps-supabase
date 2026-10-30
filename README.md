# Skills para apps com Supabase

Três skills para o [Claude Code](https://claude.com/claude-code) (e qualquer agente que leia o formato `SKILL.md`) que ensinam o agente a construir app administrativo multi-empresa com Supabase sem cair nas falhas que **não dão erro**.

Toda regra aqui saiu de incidente em produção num sistema financeiro multi-tenant real, não de documentação. O ponto em comum: no Postgres com Supabase as falhas perigosas são silenciosas. A escrita "dá certo" e não grava nada. A paginação pula e duplica linhas. O usuário vê dado de outra empresa. Nenhuma exceção, nenhum log vermelho.

| Skill | Para quê |
|---|---|
| [`supabase-app-hardening`](skills/supabase-app-hardening/SKILL.md) | RLS com as quatro políticas por tabela, gate em `SECURITY DEFINER`, `revoke` antes de `grant`, timeout de 8 segundos do `authenticated`, paginação com `.range()` que corrompe dados, receitas de verificação por personificação. |
| [`admin-app-design-system`](skills/admin-app-design-system/SKILL.md) | Camada de tokens com Tailwind e shadcn/ui, as cores semânticas que o shadcn não traz (`success`, `warning`, `info`), tema escuro, shell com sidebar e seletor de empresa. |
| [`progressive-disclosure-tables`](skills/progressive-disclosure-tables/SKILL.md) | Telas de relatório com drill-down em três níveis (número, lista que compõe o número, ficha do registro), quando usar sheet, dialog ou rota nova, filtro que vai para o banco. |

## Como instalar

**Num projeto** (vale para quem clonar o repositório do projeto):

```bash
mkdir -p .claude/skills
cp -R skills/* .claude/skills/
```

**Para você, em todos os projetos:**

```bash
mkdir -p ~/.claude/skills
cp -R skills/* ~/.claude/skills/
```

O agente carrega cada skill sozinho quando a tarefa bate com a descrição dela (os gatilhos estão em inglês e português). Para garantir, cite no `CLAUDE.md` do projeto:

```markdown
- Antes de escrever qualquer coisa de banco, leia e aplique a skill `supabase-app-hardening`.
- Antes de montar tema, shell ou telas, leia a skill `admin-app-design-system`.
- Em tela de relatório, tabela densa ou drill-down, leia a skill `progressive-disclosure-tables`.
```

## Exemplo real

O [Bússola](https://github.com/gabrielfonsecabh/bussola), sistema open source de OKR, projetos e processos com Six Sigma, foi construído com essas três skills. As migrations dele contam no cabeçalho o bug que cada uma corrigiu.

## Contribuindo

Achou outra falha silenciosa? Abra uma issue com o sintoma, a causa e a query que prova. Regra só entra com prova.

## Licença

[MIT](LICENSE). Criado por Gabriel Fonseca.
