# Decisões de modelagem – API de cadastro de usuários (MySQL)

Documento de apoio à apresentação do DER com as entidades `usuario` e `endereco`.
Perfis suportados na fase inicial: `DONO_RESTAURANTE` e `CLIENTE`.

## Resumo

| # | Tema | Decisão | Alternativa considerada |
|---|------|---------|-------------------------|
| 1 | Endereço | Tabela própria `endereco` (1:N com `usuario`) | Colunas embutidas em `usuario` |
| 2 | Tipo de perfil | `ENUM('DONO_RESTAURANTE','CLIENTE')` | Tabela `perfil` com FK |
| 3 | Senha | Somente hash em `senha_hash` (BCrypt/Argon2) | Senha em texto ou criptografia reversível (descartadas) |
| 4 | Última alteração | `TIMESTAMP ... ON UPDATE CURRENT_TIMESTAMP` | Controle apenas na aplicação ou trigger |
| 5 | Unicidade | `UNIQUE` em `email` e `login` | Validação apenas na aplicação |
| 6 | CEP | `CHAR(8)` apenas com dígitos | `VARCHAR(9)` com máscara |

---

## 1. Endereço em tabela própria

**Decisão:** o endereço (rua, número, cidade, CEP) fica na tabela `endereco`, ligada a `usuario` por `usuario_id` (FK), em relação 1:N.

**Justificativa**
- Mantém `usuario` enxuta e com responsabilidade única.
- Permite evoluir para vários endereços por usuário (entrega de clientes, filiais de restaurantes) sem alterar o schema.

**Alternativas**
- Se o escopo exigir exatamente um endereço por usuário, basta tornar `usuario_id` `UNIQUE`, o que transforma a relação em 1:1.
- Embutir as colunas em `usuario` é mais simples, porém menos flexível.

## 2. Tipo de perfil como `ENUM`

**Decisão:** a coluna `tipo` usa `ENUM('DONO_RESTAURANTE','CLIENTE')`.

**Justificativa**
- Garante no próprio banco a restrição aos dois perfis.
- Simples e suficiente para um conjunto pequeno e fixo de valores.
- Na aplicação Java, mapear com `@Enumerated(EnumType.STRING)`, evitando dependência da ordem dos valores.

**Alternativa:** se os perfis puderem crescer ou ganhar permissões, uma tabela `perfil` com FK é mais extensível.

## 3. Senha armazenada com hash

**Decisão:** guardar apenas o hash em `senha_hash VARCHAR(255)`.

**Justificativa**
- Nunca armazenar a senha em texto puro nem de forma reversível.
- Usar algoritmo próprio para senhas, como BCrypt ou Argon2.
- O BCrypt gera 60 caracteres, e `VARCHAR(255)` deixa folga para outros algoritmos.
- O sal já vai embutido no hash, dispensando coluna separada.

## 4. Data da última alteração

**Decisão:** `data_ultima_alteracao TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`.

**Justificativa**
- O banco atualiza o campo automaticamente a cada `UPDATE` na linha de `usuario`, sem depender da aplicação.

**Limitação:** alterações feitas apenas em `endereco` não atualizam esse campo. Se isso for requisito, tratar na aplicação (`@UpdateTimestamp` no Hibernate) ou com trigger.

## 5. Unicidade de e-mail e login

**Decisão:** constraints `UNIQUE` em `email` e `login` (`uk_usuario_email`, `uk_usuario_login`).

**Justificativa**
- Garante integridade mesmo com requisições concorrentes, o que a validação na aplicação sozinha não assegura.
- As constraints criam índices automaticamente, acelerando buscas por login e e-mail.
- Para unicidade de e-mail insensível a maiúsculas, normalizar para minúsculas na aplicação. A collation `utf8mb4_0900_ai_ci` do MySQL 8 já é case-insensitive por padrão.

## 6. CEP como `CHAR(8)`

**Decisão:** armazenar somente os dígitos do CEP em `CHAR(8)`.

**Justificativa**
- Tamanho fixo e sem máscara, o que facilita validação e consultas.
- A formatação (`00000-000`) fica por conta da camada de apresentação.

---

## DDL de referência

```sql
CREATE TABLE usuario (
  id                     BIGINT AUTO_INCREMENT PRIMARY KEY,
  tipo                   ENUM('DONO_RESTAURANTE','CLIENTE') NOT NULL,
  nome                   VARCHAR(150) NOT NULL,
  email                  VARCHAR(255) NOT NULL,
  login                  VARCHAR(60)  NOT NULL,
  senha_hash             VARCHAR(255) NOT NULL,
  data_ultima_alteracao  TIMESTAMP NOT NULL
                         DEFAULT CURRENT_TIMESTAMP
                         ON UPDATE CURRENT_TIMESTAMP,
  CONSTRAINT uk_usuario_email UNIQUE (email),
  CONSTRAINT uk_usuario_login UNIQUE (login)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE endereco (
  id          BIGINT AUTO_INCREMENT PRIMARY KEY,
  usuario_id  BIGINT NOT NULL,
  rua         VARCHAR(150) NOT NULL,
  numero      VARCHAR(10)  NOT NULL,
  cidade      VARCHAR(100) NOT NULL,
  cep         CHAR(8)      NOT NULL,
  CONSTRAINT fk_endereco_usuario
    FOREIGN KEY (usuario_id) REFERENCES usuario (id)
    ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
