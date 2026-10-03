# Conferência Cega

Sistema de conferência cega para o **recebimento de mercadorias** em armazéns e centros de distribuição.

## O problema

Quando um caminhão chega ao armazém, alguém precisa conferir se o que foi entregue bate com a **nota fiscal (NF)**: os produtos certos, nas quantidades certas.

Na conferência tradicional, o conferente tem a nota em mãos e já sabe que deveriam chegar, por exemplo, 48 caixas. Esse número vira uma âncora. Cansado, no fim do turno, ele tende a "enxergar" 48 caixas mesmo quando chegaram 46. É o **viés de confirmação**, e ele faz faltas e sobras passarem despercebidas.

A **conferência cega** remove a âncora: o conferente sabe **quais produtos** constam na nota, mas **não sabe as quantidades esperadas**. Ele lê o código de barras de cada item com o coletor, e quem compara o contado com o esperado é o sistema. O conferente só consegue informar o que de fato contou.

## Perfis de acesso

O sistema trabalha com **perfis**, não com pessoas. Em empresas grandes cada perfil costuma ser uma pessoa diferente; em empresas pequenas a mesma pessoa pode acumular perfis.

| Perfil | O que faz | Vê quantidades esperadas? |
|---|---|---|
| **Administrativo** | Cadastra fornecedores, produtos e notas fiscais | Sim |
| **Conferente** | Lê os códigos de barras e realiza a contagem | **Não** |
| **Supervisor** | Analisa divergências e decide o desfecho da conferência | Sim |

## Códigos de barras: EAN e DUN

Cada produto pode ser lido por dois códigos:

- **EAN-13**: identifica a **unidade**. Cada leitura soma 1 unidade.
- **DUN-14**: identifica a **caixa**. Cada leitura soma o **fator de embalagem** do produto (ex.: caixa com 12 unidades soma 12).

Por isso o cadastro de produto guarda o EAN, o DUN e o fator de embalagem.

## Fluxo principal

1. O **Administrativo** cadastra a nota fiscal: fornecedor, número e itens (produto e quantidade).
2. O **Conferente** abre a conferência da nota. Na tela ele vê a lista de produtos, **sem as quantidades**.
3. Ele lê o código de barras (EAN ou DUN) de cada item e informa a **data de validade**. Opcionalmente, pode ler um item uma vez e informar um **multiplicador** (ex.: leu 1 caixa e informou ×10).
4. Ao terminar, ele finaliza a contagem e o sistema compara o contado com a nota:
   - **Tudo confere**: a conferência é concluída.
   - **Há divergência**: o sistema informa **quais produtos** divergem, sem revelar as quantidades, e pede uma **recontagem cega** desses produtos.
5. Se a divergência persistir após a recontagem, a conferência vai para o **Supervisor**.
6. O **Supervisor** vê o esperado e o contado lado a lado, com o histórico de leituras, e decide:
   - **Aceitar a divergência**, registrando a falta ou a sobra;
   - **Mandar recontar**;
   - **Recusar a nota**.

### Status de uma conferência

```
Aberta → Em conferência → Concluída
                ↓
          Em recontagem → Concluída
                ↓
     Aguardando supervisor → Concluída com divergência
                           → Em recontagem
                           → Recusada
```

## Regras de negócio

- O conferente **nunca** vê as quantidades esperadas: nem na contagem, nem na recontagem.
- Ler um código de barras que **não pertence a nenhum produto da nota** é **bloqueado na hora**, com aviso ao conferente.
- A leitura de DUN soma o fator de embalagem do produto; a de EAN soma 1.
- Toda leitura registra a **data de validade** informada pelo conferente.
- O multiplicador é permitido, mas toda leitura fica registrada em uma **trilha de auditoria**: quem leu, quando, qual código, qual multiplicador e qual validade.
- No MVP há **uma** recontagem cega antes de a conferência ir para o supervisor.
- Só o supervisor pode aceitar divergência, mandar recontar ou recusar a nota.

## Fora do escopo (por enquanto)

- **Conferência de expedição** (saída de mercadoria), que segue um processo diferente.
- **Importação do XML da NF-e**: no MVP a nota é cadastrada manualmente. A importação de XML é a próxima fase.
- Integração com ERP.
- Controle de lote.

## Melhorias futuras

- **Etiquetas de volume**: ao concluir a conferência, imprimir uma etiqueta para cada volume ou palete conferido, com produto, quantidade e validade.
- **Ordem de serviço de armazenagem**: vincular cada etiqueta a uma O.S. que indica o endereço de destino no estoque e só é finalizada quando a mercadoria é armazenada.

## Roadmap

- [x] **Fase 0**: escopo e regras de negócio (este README)
- [ ] **Fase 1**: cadastros de fornecedores, produtos (EAN, DUN e fator de embalagem) e notas fiscais
- [ ] **Fase 2**: perfis de acesso e autorização
- [ ] **Fase 3**: conferência cega, leituras (com validade) e trilha de auditoria
- [ ] **Fase 4**: divergências, recontagem e decisão do supervisor
- [ ] **Fase 5**: importação do XML da NF-e

## Stack

- PHP 8.5 e Laravel 13
- PostgreSQL
- Tailwind CSS 4 e Vite
- Pest para testes
- Laravel Sail (Docker) para o ambiente local

## Como rodar

Pré-requisito: Docker.

```bash
git clone <url-do-repositorio>
cd conferencia-cega
cp .env.example .env
```

No `.env`, configure o banco para o PostgreSQL do Sail:

```env
DB_CONNECTION=pgsql
DB_HOST=pgsql
DB_PORT=5432
DB_DATABASE=laravel
DB_USERNAME=sail
DB_PASSWORD=password
```

Instale as dependências e suba os containers:

```bash
docker run --rm -v "$(pwd)":/var/www/html -w /var/www/html laravelsail/php85-composer:latest composer install
./vendor/bin/sail up -d
./vendor/bin/sail artisan key:generate
./vendor/bin/sail artisan migrate
./vendor/bin/sail npm install
./vendor/bin/sail npm run dev
```

Para rodar os testes:

```bash
./vendor/bin/sail artisan test
```
