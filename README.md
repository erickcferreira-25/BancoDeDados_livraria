# 📚 Banco de Dados — Livraria

Banco de dados relacional de uma livraria, feito em **MySQL 8.0**. Controla autores, categorias, livros, clientes, pedidos e os itens de cada pedido.

## Estrutura do projeto

Cada tabela tem seu próprio dump (gerado com `mysqldump`, em 29/10/2026):

| Arquivo | Tabela | Registros |
|---|---|---|
| `livraria_autor.sql` | `autor` | 6 |
| `livraria_categoria.sql` | `categoria` | 5 |
| `livraria_cliente.sql` | `cliente` | 6 |
| `livraria_livro.sql` | `livro` | 10 |
| `livraria_pedido.sql` | `pedido` | 8 |
| `livraria_item_pedido.sql` | `item_pedido` | 13 |

## Modelo de dados

```
autor ──┐                       cliente
        ├──< livro >──┐            │
categoria ┘           │            └──< pedido ──< item_pedido >── livro
                      └────────────────────────────────┘
```

```mermaid
erDiagram
    AUTOR ||--o{ LIVRO : escreve
    CATEGORIA ||--o{ LIVRO : classifica
    CLIENTE ||--o{ PEDIDO : realiza
    PEDIDO ||--|{ ITEM_PEDIDO : contem
    LIVRO ||--o{ ITEM_PEDIDO : "aparece em"
```

### Tabelas

**`autor`**
| Coluna | Tipo | Observação |
|---|---|---|
| `id_autor` | INT | PK, auto incremento |
| `nome` | VARCHAR(100) | obrigatório |
| `nacionalidade` | VARCHAR(50) | opcional |

**`categoria`**
| Coluna | Tipo | Observação |
|---|---|---|
| `id_categoria` | INT | PK, auto incremento |
| `nome` | VARCHAR(50) | obrigatório |

**`livro`**
| Coluna | Tipo | Observação |
|---|---|---|
| `id_livro` | INT | PK, auto incremento |
| `titulo` | VARCHAR(150) | obrigatório |
| `ano_publicacao` | INT | opcional |
| `preco` | DECIMAL(8,2) | obrigatório |
| `estoque` | INT | obrigatório, padrão 0 |
| `id_autor` | INT | FK → `autor` |
| `id_categoria` | INT | FK → `categoria` |

**`cliente`**
| Coluna | Tipo | Observação |
|---|---|---|
| `id_cliente` | INT | PK, auto incremento |
| `nome` | VARCHAR(100) | obrigatório |
| `email` | VARCHAR(100) | obrigatório, **único** |
| `cidade` | VARCHAR(60) | opcional |
| `data_cadastro` | DATE | opcional |

**`pedido`**
| Coluna | Tipo | Observação |
|---|---|---|
| `id_pedido` | INT | PK, auto incremento |
| `id_cliente` | INT | FK → `cliente` |
| `data_pedido` | DATE | obrigatório |
| `status` | VARCHAR(20) | Entregue, Enviado, Pendente, Cancelado |
| `forma_pagamento` | VARCHAR(20) | Pix, Cartão, Boleto |

**`item_pedido`**
| Coluna | Tipo | Observação |
|---|---|---|
| `id_item` | INT | PK, auto incremento |
| `id_pedido` | INT | FK → `pedido` |
| `id_livro` | INT | FK → `livro` |
| `quantidade` | INT | obrigatório |
| `preco_unitario` | DECIMAL(8,2) | preço praticado na venda |

> O `preco_unitario` fica no item para preservar o valor cobrado, mesmo que o `preco` do livro mude depois.

## Como restaurar

Requisitos: MySQL 8.0 ou superior.

1. Crie o banco:

```sql
CREATE DATABASE livraria CHARACTER SET utf8mb4;
```

2. Importe as tabelas **nesta ordem**, por causa das chaves estrangeiras:

```bash
mysql -u root -p livraria < livraria_autor.sql
mysql -u root -p livraria < livraria_categoria.sql
mysql -u root -p livraria < livraria_cliente.sql
mysql -u root -p livraria < livraria_livro.sql
mysql -u root -p livraria < livraria_pedido.sql
mysql -u root -p livraria < livraria_item_pedido.sql
```

Os dumps usam `DROP TABLE IF EXISTS`, então reimportar sobrescreve a tabela existente.

## Exemplos de consultas

**Valor total de cada pedido**
```sql
SELECT p.id_pedido, c.nome AS cliente, p.status,
       SUM(i.quantidade * i.preco_unitario) AS total
FROM pedido p
JOIN cliente c     ON c.id_cliente = p.id_cliente
JOIN item_pedido i ON i.id_pedido  = p.id_pedido
GROUP BY p.id_pedido, c.nome, p.status;
```

**Livros mais vendidos**
```sql
SELECT l.titulo, SUM(i.quantidade) AS unidades_vendidas
FROM item_pedido i
JOIN livro l ON l.id_livro = i.id_livro
GROUP BY l.titulo
ORDER BY unidades_vendidas DESC;
```

**Livros com seus autores e categorias**
```sql
SELECT l.titulo, a.nome AS autor, cat.nome AS categoria, l.preco, l.estoque
FROM livro l
JOIN autor a        ON a.id_autor = l.id_autor
JOIN categoria cat  ON cat.id_categoria = l.id_categoria
ORDER BY l.titulo;
```

**Faturamento por categoria (sem pedidos cancelados)**
```sql
SELECT cat.nome AS categoria,
       SUM(i.quantidade * i.preco_unitario) AS faturamento
FROM item_pedido i
JOIN pedido p       ON p.id_pedido = i.id_pedido
JOIN livro l        ON l.id_livro = i.id_livro
JOIN categoria cat  ON cat.id_categoria = l.id_categoria
WHERE p.status <> 'Cancelado'
GROUP BY cat.nome
ORDER BY faturamento DESC;
```

**Livros com estoque baixo (menos de 10 unidades)**
```sql
SELECT titulo, estoque FROM livro WHERE estoque < 10 ORDER BY estoque;
```

**Clientes por cidade**
```sql
SELECT cidade, COUNT(*) AS clientes
FROM cliente
GROUP BY cidade
ORDER BY clientes DESC;
```

## Tecnologias

- MySQL 8.0 (InnoDB, `utf8mb4`)
- Dumps gerados com `mysqldump` 10.13
