# nginx-proxy

Proxy reverso usando nginx em container Docker, responsável por receber as requisições HTTP e redirecioná-las para os serviços da aplicação.

---

## O que é o nginx?

O nginx é um servidor web de alta performance que também pode atuar como **proxy reverso**, balanceador de carga e gateway HTTP. Ele trabalha com um modelo baseado em eventos, o que o torna muito eficiente para lidar com muitas conexões simultâneas.

Internamente, o nginx possui um **processo mestre** responsável por ler e aplicar as configurações, e vários **processos worker** que de fato processam as requisições. O número de workers é configurável e pode ser ajustado automaticamente de acordo com os núcleos de CPU disponíveis.

As configurações ficam no arquivo `nginx.conf`, geralmente localizado em `/etc/nginx` ou `/usr/local/etc/nginx`.

---

## Como funciona um Proxy Reverso?

Um proxy reverso é um intermediário que fica na frente da sua aplicação. Em vez do cliente se comunicar diretamente com o servidor da aplicação, ele fala com o nginx, que repassa a requisição para o serviço correto e devolve a resposta ao cliente.

```
Cliente (navegador)
       │
       ▼
   nginx :80          ← proxy reverso
       │
       ▼
  App Node :3333      ← serviço real
```

Isso é útil para:

- Centralizar o ponto de entrada da aplicação
- Abstrair a porta real dos serviços internos
- Futuramente adicionar SSL, rate limiting, autenticação, etc.

---

## Estrutura de configuração do nginx

O nginx é configurado por **diretivas** dentro de **blocos de contexto**. Os principais contextos são:

- `http` — configurações globais de HTTP
- `server` — define um servidor virtual (porta, domínio)
- `location` — define como tratar cada rota/caminho

Exemplo de hierarquia:

```nginx
http {
    server {
        listen 80;

        location / {
            proxy_pass http://meu-servico:3333;
        }
    }
}
```

> O arquivo `default.conf` copiado pelo Dockerfile já é carregado dentro do contexto `http` pelo nginx, por isso não precisa redeclará-lo.

---

## Explicação do nosso código

### `nginx.conf`

```nginx
server {
    listen 80;

    location / {
        proxy_pass "http://172.17.0.1:3333/";
    }
}
```

| Diretiva | O que faz |
|---|---|
| `listen 80` | O nginx escuta na porta 80, que é a porta padrão HTTP |
| `location /` | Captura todas as requisições (qualquer rota) |
| `proxy_pass` | Redireciona a requisição para o endereço especificado |
| `172.17.0.1` | IP do gateway padrão da rede Docker (o host da máquina EC2) |
| `:3333` | Porta onde a aplicação Node está rodando |

O fluxo completo é:

```
Navegador → IP da EC2:80 → nginx (container) → 172.17.0.1:3333 → App Node (container)
```

### `Dockerfile`

```dockerfile
FROM nginx
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

| Instrução | O que faz |
|---|---|
| `FROM nginx` | Usa a imagem oficial do nginx como base |
| `COPY nginx.conf ...` | Substitui o arquivo de configuração padrão pelo nosso |
| `EXPOSE 80` | Documenta que o container usa a porta 80 (não publica automaticamente — isso é feito no `docker run` ou `docker-compose`) |

---

## Comandos úteis

```bash
# Build da imagem
docker build -t proxy-nginx .

# Rodar o container (porta 80 exposta)
docker run -d -p 80:80 --name proxy-container proxy-nginx

# Ver logs do nginx
docker logs proxy-container

# Parar e remover o container
docker stop proxy-container && docker rm proxy-container
```

---

## Referências

- [Nginx Beginner's Guide](https://nginx.org/en/docs/beginners_guide.html)
- [Nginx proxy_pass documentation](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass)
