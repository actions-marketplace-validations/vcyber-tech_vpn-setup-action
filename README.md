# VPN Setup Action

Estabelece um túnel VPN em runners do GitHub Actions com **healthcheck
real** — não um `sleep`.

Esta action faz parte do monorepo [**actions-rust**](https://github.com/vcyber-tech/actions-rust).
O trabalho pesado é feito pelo `vpnctl`, um binário Rust estaticamente
linkado (musl), distribuído pelas releases do monorepo.

## Por que usar esta action

A maioria dos setups de VPN em CI se parece com isso:

```yaml
- run: |
    echo "${{ secrets.VPN_CONFIG }}" > /tmp/vpn.ovpn
    sudo openvpn --config /tmp/vpn.ovpn --daemon
    sleep 20   # torcer para dar certo
```

Esta action substitui isso por algo que de fato verifica que o túnel
está funcional:

```yaml
- uses: vcyber-tech/vpn-setup-action@v1
  with:
    config: ${{ secrets.VPN_CONFIG_INLINE }}
    healthcheck-host: internal.dns.example
    healthcheck-port: '53'
```

Se o túnel não subir — ou subir mas não alcançar o host interno — o step
falha antes de você tentar fazer deploy.

## Características

- Healthcheck real — conexão TCP a um host interno, com retentativas e backoff exponencial

- Binário estático — vpnctl linkado com musl, sem dependências de runtime

- Seguro por padrão — config escrita em $RUNNER_TEMP com chmod 600, credenciais mascaradas nos logs

- Teardown idempotente — use [vpn-teardown-action](https://github.com/vcyber-tech/vpn-teardown-action) para limpar

- Verificação de checksum — todo download é validado contra SHA256

## Uso

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - name: VPN Setup
        id: vpn
        uses: vcyber-tech/vpn-setup-action@v1
        with:
          config: ${{ secrets.VPN_CONFIG_INLINE }}
          healthcheck-host: internal.dns.example
          healthcheck-port: '53'
          timeout-secs: '60'

      - name: Usar o túnel
        run: |
          echo "Túnel ativo em ${{ steps.vpn.outputs.interface }} (${{ steps.vpn.outputs.tunnel-ip }})"
          # ... comandos de deploy que dependem do túnel ...

      # Teardown sempre roda, mesmo se um step anterior falhar.
      - name: VPN Teardown
        if: always()
        uses: vcyber-tech/vpn-teardown-action@v1
```

## Inputs

| Input | Obrigatório | Padrão | Descrição |
|---|---|---|---|
| `config` | **sim** | -- | Conteúdo completo do arquivo `.ovpn`, com blocos `<ca>`, `<cert>`, `<key>` e `<tls-auth>` inline. Passe via `secrets`. |
| `provider` | não | `openvpn` | Provedor de VPN. Apenas `openvpn` é suportado. |
| `username` | não | -- | Nome de usuário, se o provedor usa auth-user-pass. |
| `password` | não | -- | Senha, se o provedor usa auth-user-pass. |
| `healthcheck-host` | não | -- | Host interno para verificar conectividade. Recomendado. |
| `healthcheck-port` | não | `443` | Porta do healthcheck. |
| `timeout-secs` | não | `60` | Timeout total, em segundos, para a conexão ser considerada bem-sucedida. |
| `interface` | não | -- | Nome esperado da interface do túnel. Auto-detectado se omitido. |
| `version` | não | -- | Versão da ferramenta Rust `vpnctl` no monorepo. Derivada da tag da action por padrão. |
| `dry-run` | não | `false` | Valida a configuração sem subir o túnel. |

## Outputs

| Output | Descrição |
|---|---|
| `interface` | Nome da interface do túnel (ex: `tun0`) |
| `tunnel-ip` | Endereço IPv4 atribuído ao túnel |

## Preparando o secret config

Seu provedor de VPN provavelmente entrega arquivos separados (ca.crt,
client.crt, client.key, ta.key, client.ovpn). Esta action espera
um único arquivo .ovpn com tudo embutido. Para construí-lo:

```yaml
# Remove referências a arquivos externos
grep -vE '^(ca|cert|key|tls-auth|tls-crypt)[[:space:]]' client.ovpn > client.inline.ovpn

# Anexa os blocos inline
{
  echo "<ca>";       cat ca.crt;     echo "</ca>"
  echo "<cert>";     cat client.crt; echo "</cert>"
  echo "<key>";      cat client.key; echo "</key>"
  echo "<tls-auth>"; cat ta.key;     echo "</tls-auth>"
} >> client.inline.ovpn
```

Depois crie um secret no repositório com o conteúdo de `client.inline.ovpn`.

## Requisitos

- Runner Linux (x86_64)

- `sudo` com NOPASSWD (padrão em runners `ubuntu-latest` do GitHub; configure explicitamente em runners self-hosted)

- Rota de rede do runner até o endpoint da VPN

## Código-fonte e issues

- Monorepo (código-fonte): [**vcyber-tech/actions-rust**](https://github.com/vcyber-tech/actions-rust)

- Issues: [abrir uma issue](https://github.com/vcyber-tech/actions-rust/issues)

- Changelog: [releases](https://github.com/vcyber-tech/actions-rust/releases)

## Licença

MIT — veja [LICENSE](./LICENSE).
