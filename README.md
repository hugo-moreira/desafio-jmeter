# Teste de performance da API de pets

Plano do Apache JMeter com o mesmo fluxo de negócio em dois modelos de carga: fechado e aberto. O arquivo do teste é `pets-modelo-fechado-aberto.jmx`.

A API exercitada é a [api-de-pets](https://github.com/tiagotaq/api-de-pets). Ela sobe em `http://localhost:3000` e já traz o usuário `tiago` com a senha `123456`.

## O que o plano cobre

Nos dois Thread Groups o caminho é o mesmo:

1. `POST /login` autentica o usuário e extrai o JWT do campo `token`.
2. `POST /pets` cadastra um pet com `Authorization: Bearer <token>`.
3. `GET /pets` consulta a lista, também autenticada.

O corpo do cadastro envia `name` e `type`. O nome usa `${__UUID}` porque a API responde `409` quando o mesmo usuário tenta cadastrar um pet com nome repetido.

## Pré-requisitos

- [API de pets](https://github.com/tiagotaq/api-de-pets) rodando na porta 3000
- Apache JMeter 5.6 ou superior

O modelo aberto usa o Open Model Thread Group, que já faz parte do JMeter desde a versão 5.5. Não é necessário instalar plugin.

## Subir a API

```bash
git clone https://github.com/tiagotaq/api-de-pets.git
cd api-de-pets
npm install
npm start
```

A documentação fica em `http://localhost:3000/api-docs`. Um login válido confirma que a API está no ar:

```bash
curl -s -X POST http://localhost:3000/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"tiago","password":"123456"}'
```

A resposta esperada é HTTP 200 com um JSON contendo `token`.

## Executar o teste

Abra `pets-modelo-fechado-aberto.jmx` na interface do JMeter para inspecionar o plano. Para a execução de carga, use o modo linha de comando, a partir da pasta `bin` do JMeter:

```bash
./jmeter -n -t pets-modelo-fechado-aberto.jmx -l resultado.jtl -e -o dashboard
```

Host, porta e credenciais estão nas variáveis do Test Plan (`protocol`, `host`, `port`, `username`, `password`). Para apontar para outra máquina, altere essas variáveis antes de rodar.

Os dois grupos disparam juntos. A agenda do modelo aberto é contada a partir do início do teste, então um grupo não espera o outro terminar.

## Carga

| | Modelo fechado | Modelo aberto |
|---|---|---|
| Elemento | Thread Group | Open Model Thread Group |
| O que está fixo | 10 usuários | Taxa de chegada |
| Subida | Ramp-up de 15 s | 1 chegada/s por 10 s, depois 3 chegadas/s por 20 s |
| Duração | 45 s | 30 s de chegadas e 5 s de pausa no fim da agenda |
| Think time | 1 s entre as requisições | Nenhum |

No modelo fechado, cada usuário espera a resposta antes da próxima chamada. Se a API ficar lenta, a vazão cai. No modelo aberto, a taxa de chegada se mantém mesmo que a resposta demore. A pausa final de 5 s existe para as iterações que ainda estão no ar concluírem login, cadastro e consulta.

## Critério de sucesso de cada requisição

| Requisição | Código esperado |
|---|---|
| `POST /login` | 200 |
| `POST /pets` | 201 |
| `GET /pets` | 200 |

O plano traz Summary Report e View Results Tree. No modo linha de comando, o resumo sai no terminal e o arquivo `resultado.jtl` guarda as amostras. A opção `-e -o dashboard` gera o relatório HTML do JMeter.
