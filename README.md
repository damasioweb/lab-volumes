# LAB 3 — Volumes e Persistência (prática do dia a dia)

> **Contexto do curso:** VM Debian com **Docker CE** instalado.  
> **Objetivo do LAB:** entender, na prática, por que **containers são descartáveis** e como **persistir dados** com volumes/bind mounts.

---

## 0) O que você vai aprender (em linguagem simples)

- **Container**: o “processo rodando agora”. Pode morrer, pode ser removido.
- **Volume**: “HD gerenciado pelo Docker” para persistir dados **fora do ciclo de vida** do container.
- **Bind mount**: “pasta do host compartilhada” com o container (muito útil em dev/lab).

**Mensagem-chave do lab:**  
Você pode destruir o container e subir outro, e os dados continuam, se estiverem em volume.

---

## 1) Pré-requisitos

1) VM Debian ligada e com internet  
2) Docker funcionando:

```bash
docker version
docker ps
```

Se `docker ps` pedir permissão, use:

```bash
sudo docker ps
```

---

## 2) Checkpoint A — Criar um volume nomeado

Crie um volume chamado `lab4_pgdata`:

```bash
docker volume create lab4_pgdata
docker volume ls | grep lab4_pgdata
```

**Sucesso:** você vê `lab4_pgdata` listado.

**Evidência (print):** saída do `docker volume ls | grep lab4_pgdata`

---

## 3) Checkpoint B — Subir Postgres usando volume (persistência real)

Suba um container do Postgres apontando o diretório de dados para o volume:

```bash
docker run -d \
  --name lab4-postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_DB=labdb \
  -v lab4_pgdata:/var/lib/postgresql/data \
  postgres:16
```

Verifique:

```bash
docker ps | grep lab4-postgres
docker logs --tail 30 lab4-postgres
```

**Sucesso:** container aparece em `docker ps` e logs não mostram erro fatal.

**Evidência (print):** `docker ps | grep lab4-postgres`

---

## 4) Checkpoint C — Criar dado no banco (para testar persistência)

Entre no container e abra o `psql`:

```bash
docker exec -it lab4-postgres psql -U postgres -d labdb
```

No prompt do `psql`, rode:

```sql
CREATE TABLE alunos (
  id SERIAL PRIMARY KEY,
  nome TEXT NOT NULL,
  criado_em TIMESTAMP DEFAULT NOW()
);

INSERT INTO alunos (nome) VALUES ('Aluno 01');
SELECT * FROM alunos;
```

Saia:

```sql
\q
```

**Sucesso:** o `SELECT` retorna pelo menos 1 linha.

**Evidência (print):** saída do `SELECT * FROM alunos;`

---

## 5) Checkpoint D — Remover o container e provar que o dado continua

Remova o container:

```bash
docker rm -f lab4-postgres
docker ps | grep lab4-postgres || echo "OK: container removido"
```

Suba outro container usando o mesmo volume:

```bash
docker run -d \
  --name lab4-postgres-2 \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_DB=labdb \
  -v lab4_pgdata:/var/lib/postgresql/data \
  postgres:16
```

Valide que os dados continuam:

```bash
docker exec -it lab4-postgres-2 psql -U postgres -d labdb -c "SELECT * FROM alunos;"
```

**Sucesso:** `Aluno 01` aparece mesmo após remover o container anterior.

**Evidência (print):** comando acima com o resultado do SELECT.

---

## 6) Bind mount (extra curto) — quando faz sentido no dia a dia

Bind mount é ótimo quando você quer editar arquivos no host e refletir no container, por exemplo:
- HTML estático
- configs em dev
- labs de “hot reload manual”

Exemplo:

```bash
mkdir -p ~/lab-bind/html
echo "Hello Bind Mount" > ~/lab-bind/html/index.html

docker run -d --name lab-bind-nginx \
  -p 8088:80 \
  -v ~/lab-bind/html:/usr/share/nginx/html:ro \
  nginx:alpine
```

Teste:

```bash
curl -s http://localhost:8088
```

Edite no host com `vim` e teste de novo:

```bash
vim ~/lab-bind/html/index.html
curl -s http://localhost:8088
```

---

---

## Evidências (para entrega)

Tire prints (ou cole o output) de:

1. **Volume criado**
   ```bash
   docker volume ls | grep lab4_pgdata
   ```

2. **Container do Postgres rodando**
   ```bash
   docker ps | grep lab4-postgres
   ```
   > Se você já removeu o primeiro container, pode mostrar o segundo:
   > ```bash
   > docker ps | grep lab4-postgres-2
   > ```

3. **Dado criado no banco (antes de remover o container)**
   - Print da saída do `SELECT` do **Checkpoint C** (dentro do `psql`):
     ```sql
     SELECT * FROM alunos;
     ```

4. **Prova de persistência (depois de remover e recriar)**
   ```bash
   docker exec -it lab4-postgres-2 psql -U postgres -d labdb -c "SELECT * FROM alunos;"
   ```

**Opcional (extra — Bind mount):**
- Print do `curl` **antes e depois** de editar o `index.html` no host:
  ```bash
  curl -s http://localhost:8088
  ```

Sugestão de pasta para organizar:

- `evidencias/` com:
  - `01-volume.png`
  - `02-ps.png`
  - `03-select-antes.png`
  - `04-select-depois.png`
  - (opcional) `05-bind-before.png`
  - (opcional) `06-bind-after.png`


## 7) Limpeza

```bash
docker rm -f lab4-postgres-2 2>/dev/null || true
docker rm -f lab-bind-nginx 2>/dev/null || true

# Se quiser apagar o volume (cuidado: apaga dados):
# docker volume rm lab4_pgdata
```

---

## Troubleshooting rápido

### 1) “permission denied” ao usar docker
Use `sudo` por enquanto:

```bash
sudo docker ps
```

### 2) Postgres reinicia em loop
Veja logs:

```bash
docker logs --tail 80 lab4-postgres
```

### 3) `psql: command not found`
Você está tentando rodar `psql` no host. Use `docker exec` (como no lab).
