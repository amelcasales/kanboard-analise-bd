# Kanboard: Etapas 2 e 3 (Mapeamento do esquema e Análise crítica)

Ambiente: MySQL 8.0.46. O banco `kanboard` tem 48 tabelas; este trabalho analisa as 15 do escopo.
Evidências coletadas em `information_schema` (KEY_COLUMN_USAGE, REFERENTIAL_CONSTRAINTS, COLUMNS).

## Etapa 2: Mapeamento do esquema

Diagrama ER: `docs/diagrama_er.png` (azul = entidades, verde = tabelas associativas, "sem FK" = relação só por convenção).

### Tabelas principais

| Tabela | Finalidade | Chaves e relacionamentos |
|---|---|---|
| **projects** | Quadro kanban; raiz de quase tudo | PK `id`. Referenciada por columns, swimlanes, tasks, project_has_users, project_has_categories, project_activities e transitions. `owner_id` aponta para users, mas sem FK. |
| **tasks** | Cartão/tarefa do quadro; tabela central | PK `id`. FKs: `project_id`, `column_id`, `swimlane_id` (todas NOT NULL). `owner_id`, `creator_id` e `category_id` sem FK. |
| **columns** | Colunas do quadro (A fazer, Em andamento...) | PK `id`. FK `project_id`. Tem `position` e `task_limit` (limite WIP). |
| **swimlanes** | Raias horizontais do quadro | PK `id`. FK `project_id`. |
| **users** | Usuários da plataforma | PK `id`, `username` UNIQUE. Referenciada por project_has_users, project_activities e transitions. |
| **project_has_users** | Associativa N:N entre projetos e usuários (membros) | FKs `project_id` → projects e `user_id` → users. |
| **comments** | Comentários em tarefas | FK `task_id` → tasks. Autor (`user_id`) sem FK. |
| **subtasks** | Subtarefas de uma tarefa | FK `task_id` → tasks. Responsável (`user_id`) sem FK. |

Demais tabelas do escopo: `task_has_links` (associativa tarefa↔tarefa via `links`, com `task_id`, `opposite_task_id`, `link_id`), `task_has_tags` (tarefa↔tag), `project_activities` (log de eventos, FKs para project, task e users), `project_has_categories` (categorias do projeto) e `transitions` (histórico de movimentação entre colunas, FKs para task, project, user, `src_column_id` e `dst_column_id`).

Cardinalidades principais: projeto 1:N colunas / raias / tarefas; coluna 1:N tarefas; raia 1:N tarefas; tarefa 1:N comentários / subtarefas; projeto N:N usuários; tarefa N:N tags; tarefa N:N tarefa (links).

## Etapa 3: Análise crítica (3 pontos)

### Ponto 1: Chaves estrangeiras: declaradas ou só por convenção?

**Evidência:** a consulta em `KEY_COLUMN_USAGE` mostra FKs declaradas em todos os vínculos "estruturais" (`tasks.project_id`, `tasks.column_id`, `tasks.swimlane_id`, `columns.project_id`, `comments.task_id`, `subtasks.task_id`, etc.). Mas vários vínculos com usuários e categorias não aparecem como FK:
- `tasks.owner_id`, `tasks.creator_id`, `tasks.category_id`
- `projects.owner_id`
- o autor de `comments` e o responsável de `subtasks`
- `tags` não declara nenhuma FK

**Por quê é problema:** o banco não impede um `owner_id` apontando para um usuário inexistente. Além disso, essas colunas têm `DEFAULT 0` e aceitam NULL (veja o ponto 2), e o valor 0 nunca poderia ser uma FK válida. A integridade fica dependente do código PHP da aplicação.

**Como verificar (evidência extra):**
```sql
SELECT COUNT(*) FROM tasks t LEFT JOIN users u ON u.id = t.owner_id
WHERE t.owner_id <> 0 AND u.id IS NULL;   -- tarefas com responsável inexistente
```

### Ponto 2: Tipos de dados e restrições

**Evidências (consulta em `information_schema.COLUMNS`):**
- **Datas em tipos diferentes:** em `tasks`, `date_creation`, `date_due`, `date_completed`, `date_started` e `date_moved` são `bigint`, mas `date_modification` é `int` (estouro em 2038). Em `projects`, `start_date` e `end_date` são `varchar(10)`: datas guardadas como texto, sem validação.
- **"Sem valor" como 0, não NULL:** `tasks.owner_id`, `creator_id`, `category_id` e `projects.owner_id` têm `DEFAULT 0`. Isso impede criar FK e mistura "não atribuído" com um id válido.
- **Tamanhos inconsistentes:** `projects.name` e `tasks.title` são `mediumtext` (até 16 MB), enquanto `columns.title` e `swimlanes.name` são `varchar(191)`. Um nome de projeto não precisa de mediumtext.
- **Booleanos em tipos diferentes:** `tasks.is_active` é `tinyint`, `swimlanes.is_active` é `int`, `users.is_active` é `tinyint(1)`. Alguns aceitam NULL (`projects.is_active`).
- **Restrições ausentes:** `users.email` não é UNIQUE (dois usuários podem ter o mesmo e-mail); `users.role` é `varchar(25)` com valor padrão `app-user`, mas sem `CHECK`/ENUM para limitar os valores.

**Impacto:** dados inconsistentes, risco de estouro de `int` em timestamps, e consultas por data mais difíceis (`varchar` não ordena/compara como data).

### Ponto 3: `ON DELETE` (o que acontece com os registros filhos)

**Evidência:** todas as FKs analisadas têm `DELETE_RULE = CASCADE` e `UPDATE_RULE = NO ACTION`.

**Consequência:** excluir um projeto apaga em cascata colunas, raias, tarefas, e, por consequência, comentários, subtarefas, links, tags, atividades (`project_activities`) e histórico (`transitions`). Excluir um usuário apaga seu histórico em `project_activities` e `transitions`, o que significa perda de trilha de auditoria.

**Ponto positivo:** não sobram registros órfãos nessas relações. **Ponto negativo:** a exclusão é irreversível e silenciosa; uma alternativa seria `SET NULL` no autor ou exclusão lógica (`is_active`).

**Como demonstrar (no banco de teste):**
```sql
START TRANSACTION;
SELECT COUNT(*) FROM tasks WHERE project_id = 1;
DELETE FROM projects WHERE id = 1;
SELECT COUNT(*) FROM tasks WHERE project_id = 1;   -- 0: cascata
ROLLBACK;
```

### Pontos extras (se sobrar tempo)
- **Padrão incomum:** `users` tem uma coluna por provedor externo (`google_id`, `github_id`, `gitlab_id`), com `gitlab_id` como `int` e os outros como `varchar(30)`; seria melhor uma tabela `user_external_accounts`. `tasks` tem 7 colunas `recurrence_*`.
- **Redundância:** `tasks` guarda `project_id`, mas `column_id` e `swimlane_id` também pertencem a um projeto; nada impede uma tarefa de apontar para uma coluna de outro projeto. `transitions` e `project_activities` repetem `project_id` além de `task_id`.
- **Charset:** verificar com `SELECT TABLE_NAME, TABLE_COLLATION FROM information_schema.TABLES WHERE TABLE_SCHEMA='kanboard';`. Colunas `varchar(191)` sugerem origem em `utf8mb4` com limite de índice antigo.
