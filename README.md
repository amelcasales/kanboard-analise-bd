# Projeto Prático: Análise do Banco de Dados do Kanboard

Plataforma sorteada: **Kanboard** (quadro kanban open source), rodando com **MySQL 8.0**.

## Pré-requisitos
- Docker e Docker Compose (v2): `docker --version` e `docker compose version`
- Portas livres: **8080** (Kanboard) e **3307** (MySQL)

## 1. Subir o ambiente
```bash
git clone <URL-DO-REPOSITORIO>
cd <PASTA-DO-REPOSITORIO>
cp .env.example .env          # ajuste as senhas se quiser
docker compose up -d
docker compose ps             # db deve estar "healthy"
```

## 2. Confirmar que o sistema funciona
1. Abra http://localhost:8080
2. Login inicial: usuário `admin`, senha `admin` (troque em *Meu perfil*)
3. Crie um projeto e uma tarefa de teste.

O Kanboard cria o esquema automaticamente no primeiro acesso (migrações em `app/Schema/Mysql.php`).

## 3. Confirmar que o banco é MySQL 8.x e que o esquema foi criado
```bash
docker compose exec db mysql -ukanboard -p kanboard -e "SELECT VERSION(); SHOW TABLES;"
```
Esperado: versão `8.0.x` e cerca de 50+ tabelas (`projects`, `tasks`, `columns`, `swimlanes`, `users`...).

Conexão por cliente externo (Workbench/DBeaver): host `127.0.0.1`, porta `3307`, usuário/senha do `.env`, banco `kanboard`.

## 4. Comandos úteis
| Ação | Comando |
|---|---|
| Parar | `docker compose stop` |
| Ver logs | `docker compose logs -f kanboard` |
| Backup do banco | `docker compose exec db mysqldump -ukanboard -p kanboard > backup.sql` |
| Resetar tudo (apaga dados!) | `docker compose down -v` |

## 5. Problemas comuns
- **Porta ocupada:** troque `8080:80` ou `3307:3306` no `docker-compose.yml`.
- **Kanboard não conecta ao banco:** aguarde o `db` ficar *healthy* e rode `docker compose restart kanboard`.
- **Erro de autenticação (caching_sha2_password):** confirme que a versão da imagem do MySQL é `8.0`.

## Fontes
- Documentação do Kanboard (Docker / banco MySQL): https://docs.kanboard.org
- Imagem oficial: https://hub.docker.com/r/kanboard/kanboard
- Manual do MySQL 8.0: https://dev.mysql.com/doc/refman/8.0/en/
