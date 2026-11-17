# MiniBench Net — client

Client di **MiniBench Net** («Valutazione indipendente di VPS e connettività»): gira su una VPS Linux, misura
con [minibench.sh](https://github.com/iacopo-cmd/minibench.sh) la CPU, la sottrazione di CPU da parte dell'hypervisor
(steal) e i core effettivi, fa ping e traceroute verso punti di interscambio, registri e reti della ricerca (mai cloud
né CDN), e manda i risultati al server di MiniBench Net. Versione 0.10.

- **Serve una chiave**, una per macchina, data da noi. Senza chiave il server non accetta niente.
- **Legge soltanto**: non cambia la configurazione della macchina, non apre porte, non installa pacchetti.
- **Privacy**: per riconoscere la macchina manda solo impronte HMAC-SHA256 firmate con un segreto che nasce sulla
  macchina e non la lascia mai (niente MAC, hostname né chiavi SSH); per la rete manda solo sì/no (IPv6, PTR, NAT, DHCP).
  Nome, provider, piano, paese e città sono facoltativi e nascosti finché non li elenchi in `PUBBLICI`.

## Installazione (da root)

```bash
D=$(mktemp -d) && cd "$D"
git clone https://github.com/iacopo-cmd/minibench-net-client.git
git clone https://github.com/iacopo-cmd/minibench.sh.git
bash minibench-net-client/installa.sh minibench.sh/minibench.sh
# poi: chiave in /etc/minibench-net/chiave (600), controlla /etc/minibench-net/config.env, prova e accendi:
python3 -I /opt/minibench-net/minibench-net.py --prova check | head
systemctl enable --now minibench-net.service minibench-net-vivo.timer
```

Istruzioni passo per passo (anche per agenti): [ISTRUZIONI-AGENTE.md](ISTRUZIONI-AGENTE.md).

Requisiti: Linux con systemd, python3 ≥ 3.9, openssl, ping, findmnt, ip; traceroute consigliato.
Licenza: MIT.
