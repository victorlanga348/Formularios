# Integração: Contratos de API REST, Transações ACID e Boletos

## 1. Visão Geral do Módulo
O `MatriculaModule` provê os endpoints REST consumidos pelo portal do estudante, orquestra simulações de valores com recálculo determinístico, executa submissões transacionais com controle estrito de idempotência e emite boletos de pagamento.

---

## 2. Contratos de API REST

### 2.1 `GET /matricula/opcoes`
Retorna as opções de cadeiras disponíveis para o estudante logado no ciclo ativo.

- **Headers:** `Authorization: Bearer <token>`
- **Response 200 OK:**
```json
{
  "cicloAtivo": 1,
  "semestreReferencia": "2026.1",
  "estudante": {
    "codigo": "20230192",
    "proximoSemestreLogico": { "ano": 2, "semestre": 1 }
  },
  "grupo1": [
    {
      "id": 101,
      "codigo": "INF201",
      "nome": "Algoritmos e Estruturas de Dados II",
      "anoCurricular": 2,
      "semestreCurricular": 1,
      "taxa": 2500.00,
      "checked": true,
      "disabled": true,
      "bloqueada": false,
      "motivoBloqueio": null,
      "precedencias": [1]
    }
  ],
  "grupo2": [
    {
      "id": 12,
      "codigo": "MAT101",
      "nome": "Álgebra Linear",
      "anoCurricular": 1,
      "semestreCurricular": 1,
      "taxa": 1800.00,
      "checked": true,
      "disabled": false,
      "bloqueada": false,
      "motivoBloqueio": null,
      "precedencias": []
    }
  ]
}
```

---

### 2.2 `POST /matricula/simular`
Calcula as taxas atualizadas e validações imediatas conforme a seleção atual do estudante.

- **Headers:** `Authorization: Bearer <token>`, `Content-Type: application/json`
- **Request Body:**
```json
{
  "cadeirasIds": [101, 12]
}
```
- **Response 200 OK:**
```json
{
  "valorTotal": 4300.00,
  "itensValidos": [
    { "cadeiraId": 101, "grupoOrigem": "REGULAR", "taxa": 2500.00 },
    { "cadeiraId": 12, "grupoOrigem": "ATRASO", "taxa": 1800.00 }
  ],
  "advertencias": []
}
```

---

### 2.3 `POST /matricula/submeter`
Efetiva a submissão com garantia de atomicidade ACID e controle estrito de concorrência/idempotência.

- **Headers:** 
  - `Authorization: Bearer <token>`
  - `Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000` (UUID v4 obrigatório)
- **Request Body:**
```json
{
  "cadeirasIds": [101, 12]
}
```
- **Fluxo de Execução Transacional:**
  1. Verificar se `IdempotencyKey` já foi processada. Se sim, retornar registro persistido em cache/banco.
  2. Submeter `cadeirasIds` ao `RulesEngineService` para validação Zero-Trust integral.
  3. Executar `prismaInternal.$transaction`:
     - Criar `InscricaoPendente` com status `PENDENTE`.
     - Criar registros associados de `InscricaoItem`.
     - Gerar código de barras e registrar `BoletoLog`.
  4. Retornar dados da inscrição e link de download do boleto.

- **Response 201 Created:**
```json
{
  "matriculaId": "b18a20d4-1a3b-483a-8b89-138407fcbe4a",
  "status": "PENDENTE",
  "valorTotal": 4300.00,
  "boletoUrl": "/matricula/b18a20d4-1a3b-483a-8b89-138407fcbe4a/boleto",
  "codigoBarras": "8467000000430000101122026010001",
  "expiraEm": "2026-09-25T23:59:59.000Z"
}
```

---

### 2.4 `GET /matricula/:id/boleto`
Retorna o buffer do PDF gerado pelo serviço de boletos.

- **Headers:** `Authorization: Bearer <token>`
- **Response 200 OK:**
  - `Content-Type: application/pdf`
  - `Content-Disposition: attachment; filename="boleto-matricula-[id].pdf"`
