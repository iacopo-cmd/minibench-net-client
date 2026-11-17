# MiniBench Net — installazione del client su una macchina nuova (istruzioni per un agente)

Queste istruzioni sono per un agente (per esempio OpenClaw) che installa il client di MiniBench Net su UNA macchina
Linux, da root. Il client misura la macchina (CPU, sottrazione di CPU da parte dell'hypervisor, rete) e manda i risultati
al server di MiniBench Net con una chiave personale della macchina. Il client legge soltanto: non cambia la
configurazione della macchina, non apre porte, non installa pacchetti da solo.

Quello che ti serve:
1. il client: il repo pubblico https://github.com/iacopo-cmd/minibench-net-client (questo file è lì dentro),
   oppure il pacchetto `minibench-net-client-<versione>.tgz` se te lo mandiamo noi;
2. il benchmark: SOLO dal repo pubblico https://github.com/iacopo-cmd/minibench.sh (nel pacchetto è già dentro);
3. la **chiave** della macchina (una riga di testo): te la diamo NOI, per un canale a parte, quando la chiedi al passo 3;
4. se serve: nome, provider, piano e minuti da evitare (chiedili prima di cominciare).

## Come comunicare con noi

- PRIMA di agire, di' cosa stai per fare (quali passi e quali comandi), poi fallo.
- Lavora e scrivi a BLOCCHI: al massimo 3 o 4 operazioni per messaggio. Dopo ogni blocco manda un messaggio con
  cosa hai fatto, l'esito e cosa farai nel blocco dopo. Non restare in silenzio fino alla fine.
- Se ti fermi (errore, dubbio, attesa della chiave), dillo subito in un messaggio: non aspettare.

## Regole

- La chiave è un SEGRETO. Non stamparla, non scriverla nei log, nella chat, nella memoria o in file diversi da
  `/etc/minibench-net/chiave`. Se ti serve controllarla, controlla solo che il file esista e abbia permessi 600.
- Una chiave vale per UNA macchina. Non riusarla su un'altra.
- Non modificare i file del pacchetto. Non toccare niente fuori da `/opt/minibench-net`, `/etc/minibench-net`,
  `/var/lib/minibench-net` e dai tre file di systemd del client.
- Se un passo fallisce, fermati e riporta il messaggio di errore esatto. Non improvvisare correzioni.
- Se sulla macchina c'è già un altro benchmark a orari fissi, chiedi i minuti da mettere in `TEST_EVITA_MINUTI`
  prima di accendere il client: due benchmark insieme si falsano a vicenda.

## 1. Controlli prima di cominciare

```bash
id -u                                   # deve essere 0 (root)
python3 --version                       # 3.9 o più recente
for c in openssl ping findmnt ip; do command -v $c >/dev/null || echo "MANCA: $c"; done
command -v traceroute >/dev/null || command -v tracepath >/dev/null || echo "MANCA: traceroute (consigliato)"
systemctl --version | head -1           # serve systemd
systemctl is-active minibench-net.service 2>/dev/null   # se dice «active» il client c'è già: fermati e chiedi
```

Se manca qualcosa, riportalo. Su Debian e Ubuntu di solito basta `apt install python3 openssl iputils-ping traceroute`,
ma installalo **solo** se te lo diciamo noi.

## 2. Scarica e installa

Dai repo pubblici:

```bash
D=$(mktemp -d) && cd "$D"
git clone https://github.com/iacopo-cmd/minibench-net-client.git
git clone https://github.com/iacopo-cmd/minibench.sh.git
bash minibench-net-client/installa.sh minibench.sh/minibench.sh
```

Oppure, se ti abbiamo mandato il pacchetto (controlla prima lo sha256 che ti diamo):

```bash
D=$(mktemp -d) && tar xzf minibench-net-client-*.tgz -C "$D"
bash "$D/minibench-net-client/installa.sh" "$D/minibench-net-client/minibench.sh"
```

`installa.sh` copia i programmi in `/opt/minibench-net`, crea `/etc/minibench-net` e `/var/lib/minibench-net`
(permessi 700), copia il modello del config e installa i file di systemd. Non accende niente.

## 3. Metti la chiave

Scrivi la chiave che ti abbiamo dato, senza mostrarla:

```bash
umask 077
printf '%s\n' "$CHIAVE" > /etc/minibench-net/chiave      # $CHIAVE: la riga che ti abbiamo mandato
chmod 600 /etc/minibench-net/chiave
unset CHIAVE
stat -c '%a %s' /etc/minibench-net/chiave                # deve dire 600 e una lunghezza > 20
```

## 4. Configura

Apri `/etc/minibench-net/config.env` e controlla che ci sia (dai repo pubblici è già così):

```
MINIBENCH_NET_URL=https://vps-a52aed3b.vps.ovh.net:60016
```

Facoltativi, solo se te li diamo noi: `NOME_HOST`, `PROVIDER`, `TIPO_HOSTING` (provider | selfhost | amico | sconosciuto),
`PIANO`, `VCPU_DICHIARATE`, `PAESE` (due lettere, es. IT), `CITTA`, `TEST_EVITA_MINUTI` (es. `18-23`).
Tutto è NASCOSTO sulla pagina pubblica finché il campo non è elencato in `PUBBLICI`: lascia `PUBBLICI=` vuoto se non
ti diciamo altro. Lascia vuoti gli `OGNI_*`: gli intervalli li decide il server.

## 5. Prova senza mandare niente

```bash
python3 -I /opt/minibench-net/minibench-net.py --prova check | head -40
```

Deve stampare un JSON con `"impronte"` (codici esadecimali: non sono i valori della macchina, sono firmati con un
segreto che resta qui) e `"qualita"`. Se stampa un errore di rete o `401`/`403`, fermati e riporta la riga.

## 6. Accendi

```bash
systemctl enable --now minibench-net.service minibench-net-vivo.timer
sleep 60
systemctl is-active minibench-net.service                # active
journalctl -u minibench-net.service --since "-2 min" --no-pager | tail -20
```

Nel registro devono comparire righe come `check: 201` e, entro qualche minuto, `target: 200`, `ping: 201`, `test: 201`.
Il primo test della CPU dura alcuni minuti e carica la macchina: è normale.

## 7. Cosa riportarci

- nome della macchina (`hostname`) e sistema (`. /etc/os-release; echo $PRETTY_NAME`);
- esito di `systemctl is-active minibench-net.service` e le ultime 20 righe del registro;
- eventuali errori, con il testo esatto.

**Non** riportare la chiave, né il contenuto di `/var/lib/minibench-net/sale` e `/var/lib/minibench-net/id_locale`.

## Se va tolto

```bash
systemctl disable --now minibench-net.service minibench-net-vivo.timer
```

I file restano dove sono; non cancellarli se non te lo chiediamo.
