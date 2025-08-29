# Projeto Flask + MySQL com Docker Compose

Este repositório contém uma aplicação **Flask** com **CRUD completo** integrada ao **MySQL**, pronta para rodar em uma VM Linux sem interface gráfica utilizando **Docker Compose**.

> Arquivos principais:
>
> - `docker-compose.yml` — orquestra serviços (db + app)
> - `app/` — Dockerfile, `requirements.txt` e `app.py` (API Flask)
> - `db/init.sql` — (opcional) script de inicialização do MySQL
>
---

## 1) Seleção e Análise do Projeto (1,0 ponto)

### Requisito
- O projeto contém **pelo menos 2 componentes**: Aplicação (Flask) e Banco de Dados (MySQL). ✅

### Arquitetura atual (descrição breve e desenho básico) — (0,5 ponto)

**Descrição:**
- Todos os serviços rodam como containers Docker no mesmo host (sua VM).
- O App (Flask) se comunica com o banco (MySQL) através da rede Docker interna.

**Diagrama (básico - arquitetura atual):**

```
[Host: sua VM]

  docker-compose
  ├─ container: app_container (Flask)  --> conecta via hostname `db`
  └─ container: mysql_container (MySQL)
```

**Pontos fortes atuais:** simples, fácil de montar, ideal para desenvolvimento local e para entregar evidências (vídeo).

**Limitações:** único host, sem balanceamento/alta-disponibilidade, sem monitoramento externo.


### Arquitetura futura (descrição breve e desenho básico) — (0,5 ponto)

**Objetivo da arquitetura futura:** tornar a solução mais resiliente, escalável e observável.

**Mudanças sugeridas:**
- Separar `app` em múltiplas réplicas (service replicas) atrás de um reverse-proxy (`nginx`) ou load-balancer.
- Mover o banco para um serviço gerenciado (RDS/Cloud SQL) ou cluster (replicação e backup automático).
- Adicionar monitoramento (Prometheus + Grafana) e logs centralizados (ELK/EFK).
- Introduzir CI/CD para deploy automático e testes.

**Diagrama (básico - arquitetura futura):**

```
[Infra (cloud / on-prem infra)]

  Internet --> [NGINX LB] --> [Replica app (Flask) x N] --> MySQL (replica set / managed)
                                         |
                                         +--> Job worker (periodic) / backup

  Observability: Prometheus / Grafana / ELK
  CI/CD: GitHub Actions / GitLab CI
```

**Benefícios:** maior disponibilidade, tolerância a falhas, capacidade de escalar horizontalmente, monitoramento e backups automáticos.

---

## 2) Análise da Arquitetura (resumo)

- **Serviços do projeto:**
  - `app` — API Flask (CRUD completo)
  - `db` — MySQL (persistência)

- **Dependências:**
  - `app` depende do `db` para operações CRUD.
  - Orquestração via Docker Compose (`depends_on` + healthcheck).

- **Estratégia de containerização:**
  - `db`: imagem oficial `mysql:8.x`, volume para persistência e script de inicialização (`/docker-entrypoint-initdb.d/init.sql`).
  - `app`: imagem baseada em `python:3.11-slim`, instalar bibliotecas via `requirements.txt`, rodar `app.py` como usuário não-root.

---

## 3) Implementação Docker Compose (resumo técnico)

Itens aplicados no projeto:
- **Definição dos serviços:** `db` (MySQL) e `app` (Flask).
- **Redes:** rede Docker bridge interna para comunicação segura entre `app` e `db`.
- **Volumes:** volume dedicado para persistência do MySQL (`db_data`).
- **Variáveis de ambiente:** configuráveis no `docker-compose.yml` (usuário, senha, database).
- **Políticas de restart:** `db` com `restart: always`; `app` com `restart: on-failure`.
- **Exposição de portas:** 3306 para MySQL (opcional), 5000 para a API.
- **Health checks:** configurado para MySQL (mysqladmin ping).
- **Usuário não-root:** `app` roda como `user: "1000:1000"`.

---

## 4) Documentação Técnica (README + comandos + deploy + troubleshooting)

### Estrutura de pastas recomendada

```
meu_projeto/
 ├─ docker-compose.yml
 ├─ README.md
 ├─ db/
 │  └─ init.sql
 └─ app/
    ├─ Dockerfile
    ├─ requirements.txt
    └─ app.py
```

### Comandos essenciais

```bash
# Subir serviços
docker compose up -d --build

# Ver status dos containers
docker compose ps

docker ps

# Ver logs
docker compose logs -f app

docker compose logs -f db

# Parar e remover
docker compose down
# Parar e remover com volumes (apaga dados do db):
docker compose down -v
```

### Processo de Deploy (passo a passo resumido)

1. Na VM, instale Docker (ex.: AlmaLinux usa `dnf`):
   ```bash
   sudo dnf update -y
   sudo dnf -y install dnf-plugins-core
   sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
   sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
   sudo systemctl enable --now docker
   sudo usermod -aG docker $USER  # opcional: logout/login
   ```

2. Criar a estrutura do projeto e salvar os arquivos (`docker-compose.yml`, `db/init.sql`, `app/*`).

3. Subir:
   ```bash
   cd ~/meu_projeto
   docker compose up -d --build
   ```

4. Aguardar `db` ficar *healthy* e testar CRUD com `curl` (exemplos abaixo).

### Testes CRUD (exemplos com `curl`)

- **Create**
```bash
curl -X POST http://localhost:5000/create \
  -H "Content-Type: application/json" \
  -d '{"nome":"André","idade":22}'
```

- **Read**
```bash
curl http://localhost:5000/read
```

- **Update**
```bash
curl -X PUT http://localhost:5000/update/1 \
  -H "Content-Type: application/json" \
  -d '{"nome":"André Antunes","idade":23}'
```

- **Delete**
```bash
curl -X DELETE http://localhost:5000/delete/1
```

### Ver operações diretamente no banco (evidência)

```bash
docker exec -it mysql_container mysql -umeu_usuario -pminha_senha meu_banco
# No prompt MySQL:
SELECT * FROM pessoas LIMIT 10;
```

### Troubleshooting básico
- **Init.sql não executou**: provavelmente o volume `db_data` já existia. Faça `docker compose down -v` e suba novamente.
- **DB não fica healthy**: ver `docker compose logs db` e validar variáveis de ambiente/credenciais.
- **App não conecta**: checar `DB_HOST` (deve ser `db`), e se o MySQL aceitou o usuário e senha.

---

## 5) Evidências para a entrega (vídeo)

No vídeo você deve mostrar:
1. `docker compose up -d --build` — iniciar os containers.
2. Logs do `db` até o `health` OK.
3. Testes CRUD via `curl` (create/read/update/delete) — mostrar resultados.
4. Acessar o MySQL via `docker exec` e rodar `SELECT` para mostrar as operações no banco.
5. Exportar JSON / gerar PNG (se aplicável) e mostrar arquivos gerados na pasta `app/`.
6. Explicação por voz de cada etapa (obrigatório segundo as observações).

---

## Observações finais
- Use imagens oficiais (`mysql`, `python`) conforme pedido.
- Certifique-se que **pelo menos 2 containers** existam (app + db) — penalidade se houver apenas um.
- Se quiser, personalize nomes (containers, banco, usuário) no `docker-compose.yml`.

---

*Arquivo gerado automaticamente. Se quiser ajustes (nome do banco, usuários, comandos adicionais ou inclusão do script `init.sql` com inserts de exemplo) eu atualizo o README conforme preferir.*
