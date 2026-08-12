# Consulta CEP — API-led Connectivity

Integração realizada para fins de conhecimento e estudo. Integração que consome o webservice da API ViaCEP, reestruturada seguindo o padrão **API-led connectivity** da MuleSoft (System / Process / Experience API).

## Arquitetura

```
Cliente (Postman/App)
        │
        ▼
Experience API  (porta 8083)  →  GET /v1/enderecos/{cep}
        │  formata a resposta para o consumidor final
        ▼
Process API     (porta 8082)  →  GET /process/endereco/{cep}
        │  valida o CEP (8 dígitos) e aplica regras de negócio
        ▼
System API      (porta 8081)  →  GET /system/cep/{cep}
        │  fala com o sistema externo e normaliza a resposta
        ▼
   ViaCEP (viacep.com.br)
```

### System API — `consultacep-system-api`
Camada mais próxima do sistema externo. Chama o ViaCEP e devolve uma resposta normalizada, escondendo os detalhes de implementação (URL, formato de resposta) do resto da organização.

- **Endpoint:** `GET /system/cep/{cep}`
- **Porta:** 8081
- Trata CEP não encontrado (`{"erro": true}` do ViaCEP → `404 CEP_NOT_FOUND`)
- Trata falha de conectividade com o ViaCEP (`503`)

### Process API — `consultacep-process-api`
Camada de orquestração e regras de negócio. Nunca fala com o ViaCEP diretamente — sempre passa pela System API.

- **Endpoint:** `GET /process/endereco/{cep}`
- **Porta:** 8082
- Valida se o CEP tem exatamente 8 dígitos numéricos (`400 CEP_INVALIDO`)
- Adiciona metadado de processamento (`consultadoEm`)
- Repassa erros da System API (404, 503)

### Experience API — `consultacep-experience-api`
Camada moldada para o consumidor final (ex: app mobile). Nunca fala com a Process API's dependências diretamente — apenas consome a Process API.

- **Endpoint:** `GET /v1/enderecos/{cep}`
- **Porta:** 8083
- Formata a resposta final em um formato amigável ao app consumidor

## Como rodar

1. Importe os 3 projetos no Anypoint Studio (`File > Import > Anypoint Studio project from File System`)
2. Rode os 3 na ordem: **System API → Process API → Experience API** (cada um sobe na sua porta, sem conflito)
3. Teste chamando a Experience API:

```
GET http://localhost:8083/v1/enderecos/01310930
```

### Exemplos de resposta

**Sucesso (200):**
```json
{
  "cep": "01310-930",
  "endereco": {
    "rua": "Avenida Paulista",
    "bairro": "Bela Vista",
    "cidade": "São Paulo",
    "estado": "SP"
  }
}
```

**CEP inválido (400):**
```json
{
  "error": "CEP_INVALIDO",
  "message": "O CEP informado deve conter exatamente 8 digitos numericos"
}
```

**CEP não encontrado (404):**
```json
{
  "error": "CEP_NOT_FOUND",
  "message": "CEP nao encontrado no ViaCEP"
}
```

## Por que separar em 3 camadas?

- **Reuso:** a System API pode ser reaproveitada por qualquer outro processo que precise de dados de CEP, sem duplicar a integração com o ViaCEP.
- **Desacoplamento:** se o ViaCEP mudar de formato ou for substituído por outro provedor, só a System API precisa mudar.
- **Responsabilidade única:** validação e regras de negócio ficam isoladas na Process API, longe de detalhes de integração e de apresentação.
- **Múltiplos canais:** novos consumidores (ex: uma Experience API para web) podem reutilizar a mesma Process API sem tocar na lógica de negócio.
