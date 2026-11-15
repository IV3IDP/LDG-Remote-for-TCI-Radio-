# LDG Remote for TCI Radio

<img width="1568" height="583" alt="image" src="https://github.com/user-attachments/assets/69fb2edc-c557-432a-bab0-ba105cda16ee" />

Controllo remoto degli accordatori automatici **LDG AT-1000ProII** e **AT-600ProII** da PC, telefono o tablet, con integrazione **TCI** per radio SDR (per esempio Hermes Lite 2) che genera la portante per l'accordatura.

*di IV3IDP* · [English](README.md)

> **Stato:** prototipo funzionante, sviluppato e provato con un **AT-600ProII**. L'AT-1000ProII usa lo stesso protocollo ma non è ancora stato provato né calibrato. Vedi [Limiti noti](#limiti-noti).

<img width="904" height="812" alt="image" src="https://github.com/user-attachments/assets/e4b9ac7d-c187-46a4-b68f-7b3633748028" />

## Caratteristiche

- **Due interfacce, un solo nucleo**
  - Finestra nativa Windows (WPF) con tema scuro.
  - Interfaccia web nel browser, utilizzabile anche da altri dispositivi della rete locale.
- **Controllo degli accordatori** per entrambi i modelli: antenna (A1/A2), Bypass, Auto, Manual, Memory Tune e Full Tune. Si possono collegare due accordatori contemporaneamente, una porta seriale ciascuno.
- **Misure in tempo reale:** frequenza (MHz), potenza diretta, riflessa, SWR, antenna attiva, modo ed esito dell'ultimo tune.
- **Barre a LED** per potenza (scala 100 W / 1000 W) e SWR, con zone verde / ambra / rossa.
- **Accordatura con portante dalla radio via TCI:** il programma avvia la portante (`tune:0,true;`), esegue il tune e spegne sempre la portante alla fine, anche in caso di errore. Riconnessione automatica al server TCI.
- **Accesso dalla rete con password** (HTTP Basic). Da `localhost` non serve password; da altri dispositivi è obbligatoria, altrimenti le richieste esterne sono bloccate.
- **Registro di cattura** di ogni byte inviato e ricevuto (`capture-*.log`), utile per il debug e per estendere il protocollo.
- Splash screen, pacchetto autoinstallante (Inno Setup), configurazione per utente.

## Come funziona

Gli accordatori hanno una **porta meter** mini-DIN a 4 poli (nata per il misuratore LDG M-1000): seriale TTL, 38400 baud, 8N1. Il programma la usa tramite un adattatore USB-TTL, decodifica il flusso di telemetria e invia comandi a un carattere.

```
Radio (TCI) <── WebSocket ──┐
                            ├── LdgCore ── USB-TTL ── AT-600ProII / AT-1000ProII
Finestra nativa / Pagina web ┘
```

## Requisiti

- Windows a 64 bit (la finestra nativa è solo Windows; la versione web gira anche su Linux e macOS dai sorgenti).
- Un **adattatore USB-seriale TTL a 5 V** (FTDI, CH340, CP210x...) per ogni accordatore e un cavo con spina mini-DIN a 4 poli.
- Una radio o un programma SDR con **server TCI** (facoltativo: serve per accordare con una portante).
- **.NET 8**, a meno di usare l'installer con il runtime incluso.

## Collegamento hardware

| Pin del tuner | Segnale | Da collegare a |
|---|---|---|
| 1 | +12 V (alimentazione del meter) | **non collegare** |
| 2 | GND | GND dell'adattatore |
| 3 | METERRX (ingresso del tuner) | **TXD** dell'adattatore |
| 4 | METERTX (uscita del tuner) | **RXD** dell'adattatore |

TX e RX sono incrociati (i nomi sono dal punto di vista del tuner). La numerazione dei pin è quella standard del mini-DIN a 4 poli; prima di saldare verifica con il tester: a tuner acceso il pin 1 deve dare circa +12 V rispetto al pin 2. Non collegare mai il pin 1 all'adattatore. Un normale cavo S-Video non è adatto perché collega tra loro alcuni conduttori.

## Installazione

### Installer

Crea uno dei due installer (servono il .NET 8 SDK e [Inno Setup 6](https://jrsoftware.org/isinfo.php)):

| Script | Risultato | Runtime |
|---|---|---|
| `build-installer.bat` | `installer\Output\LdgSuite-Setup.exe` | incluso (~80-120 MB) |
| `build-installer-light.bat` | `installer\Output\LdgSuite-Setup-Light.exe` | richiede .NET Desktop Runtime 8 e ASP.NET Core Runtime 8 (x64) |

L'installer crea i collegamenti nel menu Start, un'icona sul desktop facoltativa e una regola firewall facoltativa per la versione web. Configurazione e log stanno in `%LOCALAPPDATA%\LdgRemote`, condivisi dalla versione nativa e da quella web.

### Dai sorgenti

```
create-solution.bat     # una volta: crea LdgSuite.sln
run-native.bat          # finestra nativa
run-web.bat             # versione web, apre http://localhost:5000
```

## Configurazione

`appsettings.json` (installato: `%LOCALAPPDATA%\LdgRemote\appsettings.json`; dai sorgenti: accanto all'eseguibile):

```json
{
  "Urls": "http://localhost:5000",
  "Auth": { "User": "ldg", "Password": "" },
  "OpenBrowser": true,
  "CaptureRaw": true,
  "Tci": { "Url": "ws://localhost:50001", "Trx": 0 },
  "Tuners": [
    { "Id": "1000", "Name": "AT-1000ProII", "Port": "COM5",
      "Fwd": { "Scale": 1.0 }, "Ref": { "Scale": 1.0 } },
    { "Id": "600", "Name": "AT-600ProII", "Port": "COM6", "BandConstant": 16415,
      "Fwd": { "Scale": 29.0, "Voltage": true }, "Ref": { "Scale": 29.0, "Voltage": true } }
  ]
}
```

- **Port:** nome esatto (`COM3`, `/dev/ttyUSB0`) o parte del nome o del numero di serie dell'adattatore (per esempio `FTDI0F83`). All'avvio il programma elenca le porte seriali trovate. Con due adattatori uguali su Windows, assegna numeri COM fissi in Gestione dispositivi.
- **Calibrazione (`Fwd`, `Ref`):** `Scale` e `Voltage` (watt = (grezzo / Scale)² se `true`, grezzo × Scale se `false`), oppure `Points`, una lista di coppie `[grezzo, watt]` con interpolazione lineare a tratti, da affinare in seguito.
- **BandConstant:** MHz = `BandConstant` / valore del campo banda (vedi sotto).
- **Accesso dalla rete:** imposta `"Urls": "http://0.0.0.0:5000"` e una `Auth.Password` non vuota, poi consenti la porta nel firewall (`open-firewall.bat` o l'opzione dell'installer). La pagina è raggiungibile a `http://<indirizzo del PC>:5000`. Se la porta è occupata, viene usata la successiva libera e scritta in console.

> Il collegamento è HTTP semplice e la password viaggia in chiaro: va bene su una LAN fidata, ma **non esporre la porta a Internet**. Per l'uso da remoto usa una VPN (WireGuard, Tailscale...). Chiunque acceda può comandare gli accordatori e far trasmettere la radio.

## Note sul protocollo

Osservato su un AT-600ProII con una cattura di 5 minuti; i contributi per altri modelli sono benvenuti.

**Comandi** (un byte, un byte di risposta):

| Azione | Comando | Risposta | Note |
|---|---|---|---|
| Cambia antenna | `A` | `1` / `2` | ~90 ms |
| Memory tune | `T` | `T` SWR < 1,5 · `M` SWR 1,5–3,0 · `F` fallito | `F` dopo ~2 s se manca la RF |
| Full tune | `F` | come sopra | come sopra |
| Bypass | `P` | `P` | ~0,75 s |
| Modo Auto | `C` | `A` | documentato, non ancora catturato |
| Modo Manual | `M` | `M` | documentato, non ancora catturato |

Modi e sincronizzazione: `X` = control mode, `S` = meter mode, uno spazio sveglia il tuner. Il programma invia `X`, uno spazio, il comando e poi `S`.

**Pacchetto di telemetria** (8 byte, big-endian):

| Byte | Campo |
|---|---|
| 0–1 | potenza diretta (grezzo) |
| 2–3 | potenza riflessa (grezzo) |
| 4–5 | campo banda / contatore di frequenza |
| 6–7 | terminatore `0x3B 0x3B` (`;;`) |

- Il flusso è **continuo (~130 pacchetti/s) finché c'è RF** e solo a brevi raffiche (~0,2 s) dopo un comando altrimenti.
- Il **byte di risposta è sempre inserito tra due pacchetti**, subito dopo un `;;`; il parser separa i due.
- **Campo banda:** inversamente proporzionale alla frequenza. MHz ≈ 16415 / valore (2292 → 7,166 MHz, 1146 → 14,317 MHz, risoluzione ~3 kHz a 7 MHz e ~12 kHz a 14 MHz). `5958` significa "nessuna misura" e `65535` "non valido".
- **Scala della potenza:** con 4 W il valore diretto è circa 54–63 a seconda della banda. Il default presuppone un valore proporzionale alla tensione del rivelatore (watt = (grezzo / 29)²) e SWR = (1 + ρ) / (1 − ρ) con ρ = riflessa / diretta. È un'ipotesi da confermare con portanti ad altre potenze.

## Struttura del progetto

```
LdgCore/     libreria condivisa: protocollo seriale, parser del flusso, client TCI, calibrazione, TunerService
LdgRemote/   applicazione web ASP.NET Core (WebSocket + pagina in wwwroot)
LdgNative/   finestra nativa WPF
installer/   script Inno Setup
```

## Limiti noti

- Non ancora disponibili: i LED dei relè C/L e i pulsanti C Up/Dn, L Up/Dn, Func e Power (comandi sconosciuti).
- Un Full Tune fallisce se la radio non mantiene la portante per tutta la durata.
- La calibrazione della potenza è provvisoria; per l'AT-1000ProII i valori sono mostrati grezzi e la costante di banda non è verificata.
- La formula della frequenza è verificata su due bande (7 e 14 MHz) dell'AT-600ProII.
- La finestra nativa è solo per Windows.

## Contribuire

Il contributo più utile è un **log di cattura** (`capture-*.log`, solo timestamp e byte seriali) del tuo accordatore, con la descrizione di cosa hai fatto: frequenza e potenza della portante, comandi premuti sul pannello, modello. Con quello si possono estendere il protocollo, la calibrazione e i comandi mancanti (relè C/L). Issue e pull request sono benvenute.

## Ringraziamenti

- LDG Electronics per il documento pubblicato sul protocollo del meter M-1000.
- [LDGControl](https://github.com/frazierjason/LDGControl) per l'analisi del protocollo del meter.
- LDG-Control di GW3JVB, che ha ispirato il layout dell'interfaccia.

## Avvertenze

Progetto indipendente, **non affiliato a LDG Electronics**. LDG e i nomi AT-1000ProII / AT-600ProII sono marchi dei rispettivi proprietari. Il software può comandare la trasmissione di una radio: usalo a tuo rischio e nel rispetto delle condizioni della tua licenza.

## Licenza

