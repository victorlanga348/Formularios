# Especificação Técnica: Autenticação, Trava Financeira e Resiliência

## 1. Visão Geral do Módulo
O módulo `AuthModule` e `FinanceiroModule` é responsável pelo ciclo de vida de autenticação, derivação dinâmica de papéis (`claims`), monitoramento resiliente do banco financeiro legado e conciliação atômica pós-pagamento.

---

## 2. Contratos de Autenticação e JWT

### 2.1 Claims do Token JWT
Todo token gerado após autenticação em `POST /auth/login` transporta a seguinte estrutura de payload:

```typescript
export interface JwtPayload {
  sub: string;               // ID da sessão interna (UUID)
  codigoEstudante: string;   // Código do estudante (ex: "20230192")
  claimAcesso: ClaimAcesso;  // "SOMENTE_BOLETO" | "LIVRE"
  iat?: number;
  exp?: number;
}

export enum ClaimAcesso {
  SOMENTE_BOLETO = "SOMENTE_BOLETO",
  LIVRE = "LIVRE"
}
```

### 2.2 Regras de Acesso por Claim
- **`SOMENTE_BOLETO`**: O estudante possui saldo devedor > 0 no banco legado ou o sistema está em fallback de proteção financeira. O acesso a rotas de `/matricula/*` é sumariamente rejeitado (HTTP 403 Forbidden). O estudante é confinado a `/financeiro/pendencia` e telas de liquidação.
- **`LIVRE`**: O estudante possui saldo devedor zerado. O acesso ao fluxo de inscrição acadêmica é liberado.

---

## 3. Prioridade 0: Trava Financeira & Resiliência (Circuit Breaker)

### 3.1 Arquitetura do Circuit Breaker (Opossum)
A consulta ao banco financeiro legado (`PrismaExternalClient`) é envelopada por uma instância do Opossum com as seguintes configurações:
- **Timeout:** 3000ms
- **Error Threshold Percentage:** 50%
- **Reset Timeout:** 15000ms

### 3.2 Lógica Algorítmica de Fail-Closed e Reconciliação
```typescript
async function verificarTravaFinanceira(codigoEstudante: string): Promise<ClaimAcesso> {
  try {
    const saldoExterno = await circuitBreaker.fire(codigoEstudante); // Consulta PrismaExternal
    
    // Atualização do cache local
    await prismaInternal.pendenciaCache.upsert({
      where: { estudanteCodigo: codigoEstudante },
      create: { 
        estudanteCodigo: codigoEstudante, 
        ultimoSaldoConhecido: saldoExterno, 
        emFallback: false 
      },
      update: { 
        ultimoSaldoConhecido: saldoExterno, 
        emFallback: false 
      }
    });

    if (saldoExterno > 0) {
      return ClaimAcesso.SOMENTE_BOLETO;
    }

    // Reconciliação Reativa no Login: Efetivação Automática Pós-Pagamento
    await prismaInternal.inscricaoPendente.updateMany({
      where: { 
        estudanteCodigo: codigoEstudante, 
        status: InscricaoStatus.PENDENTE 
      },
      data: { 
        status: InscricaoStatus.EFECTIVADA 
      }
    });

    return ClaimAcesso.LIVRE;
  } catch (error) {
    // Fallback Defensivo Fail-Closed: Nunca liberar por omissão
    const cache = await prismaInternal.pendenciaCache.findUnique({ 
      where: { estudanteCodigo: codigoEstudante } 
    });
    
    if (!cache || cache.ultimoSaldoConhecido.toNumber() > 0) {
      return ClaimAcesso.SOMENTE_BOLETO;
    }
    
    // Em caso de falha externa sem garantia absoluta, fail-closed por segurança
    return ClaimAcesso.SOMENTE_BOLETO;
  }
}
```

---

## 4. Prioridade 1: Validação de Calouro
A inscrição semestral online não é aplicável a estudantes no seu primeiro semestre curricular:

```typescript
if (aluno.anoCurricularAtual === 1 && aluno.semestreCurricularAtual === 1) {
  throw new ForbiddenException("Matrícula de calouros é manual. Dirija-se à secretaria acadêmica.");
}
```

---

## 5. Endpoints REST do Módulo

### `POST /auth/login`
- **Request Body:**
  ```json
  {
    "codigo": "20230192",
    "senha": "senhaSeguraHash"
  }
  ```
- **Response 200 OK:**
  ```json
  {
    "accessToken": "eyJhbGciOiJIUzI1NiIsIn...",
    "claimAcesso": "LIVRE",
    "aluno": {
      "codigo": "20230192",
      "anoCurricular": 2,
      "semestreCurricular": 1
    }
  }
  ```
- **Response 403 Forbidden (Calouro):**
  ```json
  {
    "statusCode": 403,
    "message": "Matrícula de calouros é manual. Dirija-se à secretaria acadêmica."
  }
  ```

### `GET /financeiro/pendencia`
- **Headers:** `Authorization: Bearer <token>`
- **Response 200 OK:**
  ```json
  {
    "estudanteCodigo": "20230192",
    "saldoDevedor": 4500.00,
    "emFallback": false,
    "boletoLiquidacaoUrl": "/financeiro/boletos/liquidacao-20230192.pdf"
  }
  ```
